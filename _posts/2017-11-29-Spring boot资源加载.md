---
layout: post
title:  "Spring Boot资源加载"
date:   2017-11-29
tags:
    - Spring
    - Spring Boot
---

## 0x01 fat-jar

```xml
<plugin>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-maven-plugin</artifactId>
</plugin>
```
在使用maven打包的时候，在配置文件加入上面这段配置会开启spring boot的打包。
输入如下：

		[INFO] --- spring-boot-maven-plugin:1.5.6.RELEASE:repackage (default) @ kof-logic-server ---

在正常执行maven的打包插件后，Spring-boot会将maven打的包重新进行一次repackage的过程。打成spring boot特有的fat-jar格式。<br>
目录结构大致如下:

		BOOT-INF/
		  classes/
			target下的东西
		  lib/
			应用的依赖
		META-INF/
		  MANIFEST.MF
		  maven/
		org/
		  spring boot loader代码


## 0x02 spring boot包内的资源加载
现在有这样的需求，maven copy-resources阶段把资源copy到target下，然后在运行时去加载。<br>
使用ClassPathResource是不行的，需要使用UrlResource来加载。demo如下：

```java
	public void init() {
		beanName2Cfgs = fetch(MAIN_GBEANS_FILE_NAME)
			.filter(e -> e.getAttribute("genxml").equals("server"))
			.map(this::parseResource)
			.filter(r -> r != null)
			.map(this::loadResource)
			.collect(Collectors.toConcurrentMap(Pair::getKey, Pair::getValue));
		
		if (!isLoadSuccess) {
			throw new RuntimeException("ConfigManager加载失败!!！");
		}
	}
	
	private Stream<Element> fetch(String gbeansName) {
		Resource res = newResource(gbeansName);
		if (res == null) {
			return Stream.empty();
		}
		
		DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
		try {
			DocumentBuilder builder = factory.newDocumentBuilder();	
			Document doc = builder.parse(res.getInputStream());

			// load all gbeans
			Stream<Element> stream = Stream.empty();
			NodeList nl = doc.getElementsByTagName("xi:include");
			for (int i = 0; i < nl.getLength(); i++) {
				Element e =(Element) nl.item(i);
				stream = Stream.concat(fetch(e.getAttribute("href")), stream);
			}
			
			// load configuration
			XPathFactory xFactory = XPathFactory.newInstance();
			XPath xpath = xFactory.newXPath();
			Object result = xpath.evaluate("//bean", doc, XPathConstants.NODESET);
			nl = (NodeList) result;
			for (int i = 0; i < nl.getLength(); i++) {
				stream = Stream.concat(Stream.of((Element)nl.item(i)), stream);
			}
			return stream;
		} catch (Exception e) {
			LOGGER.error(e.getMessage());
			isLoadSuccess = false;
			return Stream.empty();
		}
	}
	
	private Resource parseResource(Element e) {
		String fullName = e.getAttribute("name") + ".xml";
		Node parent = e.getParentNode();
		while (parent != null && parent instanceof Element) {
			Element pe = (Element) parent;
			fullName = pe.getAttribute("name") + "." + fullName;
			parent = parent.getParentNode();
		}
		
		fullName = fullName.startsWith(CONFIG_PREFIX) ? fullName : CONFIG_PREFIX + fullName;
		return newResource(fullName);
	}
	
	private Pair<String, Object> loadResource(Resource res) {
		try {
			...
			return new Pair<String, Object>(key, value);
		} catch (Exception e) {
			LOGGER.error(e.getMessage());
			isLoadSuccess = false;
			return new Pair<String, Object>("", "");
		}
	}
	
	private Resource newResource(String fileName) {
		try {
			return new UrlResource("classpath:/" + fileName);
		} catch (MalformedURLException e) {
			LOGGER.error("初始化" + fileName + "的UrlResource时失败！！");
			isLoadSuccess = false;
			return null;
		}
	}
```
上面的demo就是从spring boot fat-jar中的main.xml开始，先遍历出include的所有xml中的bean元素，然后按照一定的规则拼接成文件名，最后再去加载最终资源。


## 0x03 url的格式

* file:./custom-config/
* classpath:custom-config/
* file:./config/
* file:./
* classpath:/config/
* classpath:/
