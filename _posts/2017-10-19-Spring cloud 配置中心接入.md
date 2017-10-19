---
layout: post
title:  "Spring cloud 配置中心接入"
date:   2017-10-19
categories: Spring
---

## 0x01 简介
pring Cloud Config支持在Git, SVN和本地存放配置文件，使用Git或者SVN存储库可以很好地支持版本管理，Spring默认配置是使用Git存储库。


## 0x02 代码
```java
@EnableConfigServer
@SpringCloudApplication
public class ConfigServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
    
}
```

## 0x03 依赖
```xml
	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-config-server</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-eureka</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-hystrix</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-config-monitor</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-bus-amqp</artifactId>
		</dependency>
		<dependency>
			<groupId>org.jolokia</groupId>
			<artifactId>jolokia-core</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>

```

## 0x03 git
```yml
server:
  port: 8003

spring:
  rabbitmq:
    host: kof-ms.sys.ledo.com
    username: guest
    password: guest
    virtual-host: /
    listener:
      retry:
        enabled: true
    template:
      retry:
        enabled: true
  cloud:
    config:
      server:
        git:
          uri: git的地址 
```				
## 0x04 svn

#### 其他依赖
```xml
	<dependency>
		<groupId>org.tmatesoft.svnkit</groupId>
		<artifactId>svnkit</artifactId>
	</dependency>
```

#### 配置
```yml
server:
  port: 8003

spring:
  rabbitmq:
    host: kof-ms.sys.ledo.com
    username: guest
    password: guest
    virtual-host: /
    listener:
      retry:
        enabled: true
    template:
      retry:
        enabled: true
  profiles:
    active: subversion
  cloud:
    config:
      enabled: true
      server:
        svn:
          uri: svn地址
          username: 'xxx'
          password: 'xxx'
        default-label: trunk
  application:
    name: 'kof-config-server'
```

## 0x05 native

```yml
server:
  port: 8003

spring:
  rabbitmq:
    host: kof-ms.sys.ledo.com
    username: guest
    password: guest
    virtual-host: /
    listener:
      retry:
        enabled: true
    template:
      retry:
        enabled: true
  profiles:
    active: subversion
  cloud:
    config:
      enabled: true
      server:
        native:
          searchLocations: 本地仓库地址
  application:
    name: 'kof-config-server'
```