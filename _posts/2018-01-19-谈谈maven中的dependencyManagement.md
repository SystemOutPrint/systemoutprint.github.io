---
layout: post
title:  "谈谈Maven中的dependencyManagement"
date:   2018-01-19
author:       "CaiJiahe"
header-img:   "img/tag-bg.jpg"
header-mask:  0.3
catalog:      true
multilingual: true
tags:
    - Maven
---

## 0x01 dependencyManagement
Maven使用dependencyManagement元素来提供了一种管理依赖版本号的方式。通常会在一个组织或者项目的最顶层的父POM中看到dependencyManagement元素。使用pom.xml中的dependencyManagement元素能让
所有在子项目中引用一个依赖而不用显式的列出版本号。Maven会沿着父子层次向上走，直到找到一个拥有dependencyManagement元素的项目，然后它就会使用在这个dependencyManagement元素中指定的版本号。

## 0x02 问题
在dependencyManagement中import一个pom，无法修改该pom依赖中的版本号。<br>
例：<br>
```xml
<dependencyManagement>
    <dependencies>
      <dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-dependencies</artifactId>
		<version>Dalston.SR4</version>
		<type>pom</type>
		<scope>import</scope>
      </dependency>
    </dependencies>
</dependencyManagement>
```
现在我想修改spring-cloud-dependencies中的spring-cloud-consul-dependencies的版本号。如果不去修改spring-cloud-dependencies是无法生效的。
```xml
<dependencyManagement>
    <dependencies>
      <dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-dependencies</artifactId>
		<version>Dalston.SR4</version>
		<type>pom</type>
		<scope>import</scope>
		<exclusions>
			<exclusion>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-consul-dependencies</artifactId>
			</exclusion>
		</exclusions>
      </dependency>
      <dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-consul-dependencies</artifactId>
		<version>1.3.0.RELEASE</version>
		<type>pom</type>
		<scope>import</scope>
	  </dependency>
    </dependencies>
</dependencyManagement>
```
上面的这种示例是错误的。附一个maven的Improvement。<br>
[Maven/MNG-5600 Dependency management import should support exclusions.](https://issues.apache.org/jira/browse/MNG-5600)<br>
静静等待新版本的maven更新这个功能。Orz...

## 0x03 解决办法
在使用spring-cloud-consul-dependencies的地方声明dependencyManagement去覆盖父pom的版本。<br>
例：<br>
```xml
<dependencyManagement>
	<dependencies>
	<dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-consul-dependencies</artifactId>
		<version>1.3.0.RELEASE</version>
		<type>pom</type>
		<scope>import</scope>
	  </dependency>
	 </dependencies>
</dependencyManagement>
```
