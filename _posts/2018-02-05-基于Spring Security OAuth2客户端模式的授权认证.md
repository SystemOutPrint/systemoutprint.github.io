---
layout: post
title:  "基于Spring Security OAuth2客户端模式的授权认证"
date:   2018-02-05
author:       "CaiJiahe"
header-img:   "img/tag-bg.jpg"
header-mask:  0.3
tags:
    - Spring
    - Spring Security
    - OAuth2
---

## 0x01 目的
网上关于使用Spring Security OAuth2的客户端模式实现的授权认证服务的资料比较少。寥寥几篇博文也只是使用了简单的InMemory，而在生产环境中，我们要使用mysql这样的持久化的存储来存储一些授权信息。
所以本文实现一个基于OAuth2客户端模式的授权认证服务器，ClientDetails存在mysql中，Token的信息存在redis中。

## 0x02 application.xml配置
```yml
server:
  port: 8080

spring:
  datasource:
    url: jdbc:log4jdbc:mysql://127.0.0.1:3306/kof?useUnicode=true&characterEncoding=utf-8
    username: xxx
    password: xxx
    driver-class-name: net.sf.log4jdbc.DriverSpy
  redis:
    host: 127.0.0.1
    database: 0

security:
  oauth2:
    resource:
      filter-order: 3

flyway:
  enabled: true
  baseline-on-migrate: true
  validate-on-migrate: false
```
配置中配置了mysql的datasource，还有redis的host。使用flyway来维护schema的定义。

## 0x03 数据库表定义
[mysql表的定义](https://github.com/spring-projects/spring-security-oauth/blob/master/spring-security-oauth2/src/test/resources/schema.sql)直接下载下来不能直接在mysql中创建，因为mysql不支持LONGVARBINARY，所以要修改为BLOB才能用。为此我提交了一个[Pull request](https://github.com/spring-projects/spring-security-oauth/pull/1282)。<br>

## 0x04 AuthServer的Java Config
```java
@Configuration
@EnableAuthorizationServer
public class AuthorizationServerConfiguration extends AuthorizationServerConfigurerAdapter {

	@Autowired
	AuthenticationManager authenticationManager;

	@Autowired
	RedisConnectionFactory redisConnectionFactory;

	@Autowired
	DataSource dataSource;

	@Bean
	public UserDetailsService userDetailsService(ClientDetailsService cds) {
		return new ClientDetailsUserDetailsService(cds);
	}

	@Override
	public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
		clients.jdbc(dataSource);
	}

	@Override
	public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
		endpoints.tokenStore(new RedisTokenStore(redisConnectionFactory))
				.authenticationManager(authenticationManager);
	}

}
```
AuthServer的配置：
* 注入了DataSource和RedisConnectionFactory。
* ClientDetailsService配置成JdbcClientDetailsService。
* 替换UserDetailsService为ClientDetailsUserDetailsService。
* 设置tokenStore为RedisTokenStore。

## 0x05 OAuth2客户端模式验证认证
```java
// ClientCredentialsTokenEndpointFilter.attemptAuthentication
// 过滤访问/oauth/token的请求。
@Override
public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response)
		throws AuthenticationException, IOException, ServletException {
	...
	
	UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(clientId,
			clientSecret);

	return this.getAuthenticationManager().authenticate(authRequest);
}

// ProviderManager.authenticate
// 找到相应的AuthenticationProvider来处理该认证
public Authentication authenticate(Authentication authentication)
			throws AuthenticationException {
	...
	
	for (AuthenticationProvider provider : getProviders()) {
		if (!provider.supports(toTest)) {
			continue;
		}
		result = provider.authenticate(authentication);
	}
	
	...

	if (result != null) {
		eventPublisher.publishAuthenticationSuccess(result);
		return result;
	}
	
	...
}

// AbstractUserDetailsAuthenticationProvider.authenticate
// 加载user信息，然后去认证。
public Authentication authenticate(Authentication authentication)
			throws AuthenticationException {
	...
	
	if (user == null) {
		try {
			user = retrieveUser(username,
					(UsernamePasswordAuthenticationToken) authentication);
		} catch (UsernameNotFoundException notFound) {
			...
		}
		...
	}
	
	try {
		preAuthenticationChecks.check(user);
		additionalAuthenticationChecks(user,
				(UsernamePasswordAuthenticationToken) authentication);
	}
	catch (AuthenticationException exception) {
		...
	}

	...

	return createSuccessAuthentication(principalToReturn, authentication, user);
}

// DaoAuthenticationProvider.retrieveUser
// 调用UserDetailsService去加载用户，这里可以使用InMemory或者Jdbc的实现。
protected final UserDetails retrieveUser(String username,
			UsernamePasswordAuthenticationToken authentication)
			throws AuthenticationException {
	UserDetails loadedUser;
	try {
		loadedUser = this.getUserDetailsService().loadUserByUsername(username);
	}
	...
	return loadedUser;
}
	
// JdbcClientDetailsService.loadClientByClientId
// 使用ClientDetailsRowMapper反序列化ResultSet为ClientDetails。
// 然后在ClientDetailsUserDetailsService中将ClientDetails适配为UserDetails，其实就是将clientSecure作为UserDetail的password。
public ClientDetails loadClientByClientId(String clientId) throws InvalidClientException {
	ClientDetails details;
	try {
		details = jdbcTemplate.queryForObject(selectClientDetailsSql, new ClientDetailsRowMapper(), clientId);
	}
	catch (EmptyResultDataAccessException e) {
		throw new NoSuchClientException("No client with requested id: " + clientId);
	}

	return details;
}

// DaoAuthenticationProvider.additionalAuthenticationChecks
// 简单判断是否相等，不相等则抛异常。
protected void additionalAuthenticationChecks(UserDetails userDetails,
			UsernamePasswordAuthenticationToken authentication)
			throws AuthenticationException {
	...

	String presentedPassword = authentication.getCredentials().toString();

	if (!passwordEncoder.isPasswordValid(userDetails.getPassword(),
			presentedPassword, salt)) {
		logger.debug("Authentication failed: password does not match stored value");

		throw new BadCredentialsException(messages.getMessage(
				"AbstractUserDetailsAuthenticationProvider.badCredentials",
				"Bad credentials"));
	}
}

```

## 0x06 OAuth2客户端模式创建Token
```java
// TokenEndpoint.postAccessToken
@RequestMapping(value = "/oauth/token", method=RequestMethod.POST)
public ResponseEntity<OAuth2AccessToken> postAccessToken(Principal principal, @RequestParam
			Map<String, String> parameters) throws HttpRequestMethodNotSupportedException {
	...

	String clientId = getClientId(principal);
	ClientDetails authenticatedClient = getClientDetailsService().loadClientByClientId(clientId);
	TokenRequest tokenRequest = getOAuth2RequestFactory().createTokenRequest(parameters, authenticatedClient);

	...

	OAuth2AccessToken token = getTokenGranter().grant(tokenRequest.getGrantType(), tokenRequest);
	if (token == null) {
		throw new UnsupportedGrantTypeException("Unsupported grant type: " + tokenRequest.getGrantType());
	}
	return getResponse(token);
}

// CompositeTokenGranter.grant
public OAuth2AccessToken grant(String grantType, TokenRequest tokenRequest) {
	for (TokenGranter granter : tokenGranters) {
		OAuth2AccessToken grant = granter.grant(grantType, tokenRequest);
		if (grant!=null) {
			return grant;
		}
	}
	return null;
}

// ClientCredentialsTokenGranter.grant
@Override
public OAuth2AccessToken grant(String grantType, TokenRequest tokenRequest) {
	OAuth2AccessToken token = super.grant(grantType, tokenRequest);
	...
	return token;
}

// AbstractTokenGranter.grant
public OAuth2AccessToken grant(String grantType, TokenRequest tokenRequest) {
	if (!this.grantType.equals(grantType)) {
		return null;
	}
	
	String clientId = tokenRequest.getClientId();
	ClientDetails client = clientDetailsService.loadClientByClientId(clientId);
	validateGrantType(grantType, client);
	
	logger.debug("Getting access token for: " + clientId);

	return getAccessToken(client, tokenRequest);
}

// AbstractTokenGranter.getAccessToken
protected OAuth2AccessToken getAccessToken(ClientDetails client, TokenRequest tokenRequest) {
	return tokenServices.createAccessToken(getOAuth2Authentication(client, tokenRequest));
}

// DefaultTokenServices.createAccessToken
// 创建个token返回
@Transactional
public OAuth2AccessToken createAccessToken(OAuth2Authentication authentication) throws AuthenticationException {

	OAuth2AccessToken existingAccessToken = tokenStore.getAccessToken(authentication);
	OAuth2RefreshToken refreshToken = null;
	if (existingAccessToken != null) {
		if (existingAccessToken.isExpired()) {
			if (existingAccessToken.getRefreshToken() != null) {
				refreshToken = existingAccessToken.getRefreshToken();
				// The token store could remove the refresh token when the
				// access token is removed, but we want to
				// be sure...
				tokenStore.removeRefreshToken(refreshToken);
			}
			tokenStore.removeAccessToken(existingAccessToken);
		}
		else {
			// Re-store the access token in case the authentication has changed
			tokenStore.storeAccessToken(existingAccessToken, authentication);
			return existingAccessToken;
		}
	}

	// Only create a new refresh token if there wasn't an existing one
	// associated with an expired access token.
	// Clients might be holding existing refresh tokens, so we re-use it in
	// the case that the old access token
	// expired.
	if (refreshToken == null) {
		refreshToken = createRefreshToken(authentication);
	}
	// But the refresh token itself might need to be re-issued if it has
	// expired.
	else if (refreshToken instanceof ExpiringOAuth2RefreshToken) {
		ExpiringOAuth2RefreshToken expiring = (ExpiringOAuth2RefreshToken) refreshToken;
		if (System.currentTimeMillis() > expiring.getExpiration().getTime()) {
			refreshToken = createRefreshToken(authentication);
		}
	}

	OAuth2AccessToken accessToken = createAccessToken(authentication, refreshToken);
	tokenStore.storeAccessToken(accessToken, authentication);
	// In case it was modified
	refreshToken = accessToken.getRefreshToken();
	if (refreshToken != null) {
		tokenStore.storeRefreshToken(refreshToken, authentication);
	}
	return accessToken;
}
 
```
创建token的过程的过程很简单，先从TokenStore中查询是否当前client_id是否被认证过，如果已经认证过并且未过期，则直接返回认证的token。否则尝试去创建refresh token，然后去创建token存储返回。<br>
access token和refresh token的过期时间分别对应oauth_client_details的access_token_validity和refresh_token_validity，单位是秒。
