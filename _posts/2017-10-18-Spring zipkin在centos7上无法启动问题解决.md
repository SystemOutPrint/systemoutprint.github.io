---
layout: post
title:  "Spring zipkin在centos7上无法启动问题解决方法"
date:   2017-10-18
categories: Spring
---

## 0x01 问题
zipkin程序，在windows上运行一切OK，但是在CentOS7上启动就会报异常

		Caused by: org.springframework.beans.BeanInstantiationException: Failed to instantiate [org.springframework.boot.web.filter.OrderedHttpPutFormContentFilter]: Factory method 'httpPutFormContentFilter' threw exception; nested exception is java.lang.NoClassDefFoundError: Could not initialize class com.fasterxml.jackson.databind.ObjectMapper
		
## 0x02 环境
__操作系统__: CentOS7<br>
__JRE版本__: JRE 1.8.0_144/1.8.0_131<br>
__Spring Boot__: 1.5.6 RELEASE<br>
__Spring Cloud__: Dalston.SR4/Dalston.SR3
		
## 0x03 排查方法
打开Spring Boot的fat jar，发现是有这个类的。<br>
因为Windows上运行OK，Linux上失败，所以考虑是JRE导致的，Linux上是1.8.0_131，Windows是1.8.0_144，所以升级了下Linux上的JRE版本，但是没有效果。<br>
在google上搜索相关问题发现类似问题([传送门](https://github.com/spring-projects/spring-boot/issues/9408))，根据题主的排查方法，可以看到本项目的依赖版本。

		mvn dependency:tree | grep 'jackson' 
			`[INFO] +- com.fasterxml.jackson.dataformat:jackson-dataformat-xml:jar:2.8.9:compile
			[INFO] |  +- com.fasterxml.jackson.core:jackson-core:jar:2.8.9:compile
			[INFO] |  +- com.fasterxml.jackson.core:jackson-annotations:jar:2.8.0:compile
			[INFO] |  +- com.fasterxml.jackson.core:jackson-databind:jar:2.8.9:compile

可以观察到jackson-annotations的版本是2.8.0，比其他的版本低，可能是旧版本和新版本的类冲突了。所以尝试排除databind的依赖jackson-annotations，然后显示导入。

		<dependency>
			<groupId>com.fasterxml.jackson.core</groupId>
			<artifactId>jackson-databind</artifactId>
			<exclusions>
				<exclusion>
					<groupId>com.fasterxml.jackson.core</groupId>
					<artifactId>jackson-annotations</artifactId>
				</exclusion>
			</exclusions>
		</dependency>
		<dependency>
			<groupId>com.fasterxml.jackson.core</groupId>
			<artifactId>jackson-annotations</artifactId>
		</dependency>
		
问题解决。