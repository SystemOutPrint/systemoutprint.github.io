---
layout: post
title:  "Maven antrun的故事"
date:   2017-11-08
author:       "CaiJiahe"
header-img:   "img/tag-bg.jpg"
header-mask:  0.3
catalog:      true
multilingual: true
tags:
    - Maven
    - Ant
---

## 0x01 java target

#### 问题：
老项目使用ant编译，生成协议文件的jar包不依赖其他jar包。所以使用的是java -jar jar_location，main class写在MANIFEST中。
```xml
<java jar="jar_location" fork="yes">
	<arg value="xmlgen" />
	<arg value="-java" />
</java>
```
这么写是OK的，但是在修改了这个jar包，增加一些依赖后，蛋疼的事发生了，每次运行都会报ClassNoDef的异常。

#### 解决办法
类似于java -cp:xxx main_class_name
```xml
<java classname="main_class_location" fork="yes">
	<classpath>
		<pathelement location="${jar_location}"/>
		<pathelement location="${dep_location}"/>
	</classpath>
	<arg value="xmlgen" />
	<arg value="-java" />
</java>
```

## 0x02 Referencing the Maven Classpaths

	Note: the old format "maven.dependency.groupId.artifactId[.classifier].type.path" has been deprecated and should no longer be used.
	
现在的格式是 
	
	groupId:artifactId:type[:classifier]

