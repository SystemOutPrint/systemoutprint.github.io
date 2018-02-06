---
layout: post
title:  "Spring Boot配置使用优先级"
date:   2017-10-16
author:       "CaiJiahe"
header-img:   "img/tag-bg.jpg"
header-mask:  0.3
catalog:      true
multilingual: true
tags:
    - Spring
    - Spring Boot
---

配置使用的优先级从高到低的顺序:
* 命令行参数。
* 通过 System.getProperties() 获取的 Java 系统参数。
* 操作系统环境变量。
* 从 java:comp/env 得到的 JNDI 属性。
* 通过 RandomValuePropertySource 生成的“random.*”属性。
* 应用 Jar 文件之外的属性文件(application.properties)。
* 应用 Jar 文件内部的属性文件(application.properties)。
* 在应用配置 Java 类（包含“@Configuration”注解的 Java 类）中通过“@PropertySource”注解声明的属性文件。
* 通过“SpringApplication.setDefaultProperties”声明的默认属性。