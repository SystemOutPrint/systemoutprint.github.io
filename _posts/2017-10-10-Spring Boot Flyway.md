---
layout: post
title:  "Spring Boot Flyway"
date:   2017-10-10
author:       "CaiJiahe"
header-img:   "img/tag-bg.jpg"
header-mask:  0.3
catalog:      true
multilingual: true
tags:
    - Spring
    - Spring Boot
    - Flyway
---

## 0x01 简介
开发人员在合作的时候经常遇到以下场景：<br>
* 开发人员A在自己的本地数据库做了一些表结构的改动，并根据这些改动调整了DAO层的代码，然后将代码上传到svn或git等版本控制服务器上。此时如果开发人员B拉取了A的代码改动，在运行项目的时候很可能会报错，因为B的本地SQL数据库并没有修改。
* 在项目上线的时候，当服务器拉取的版本控制服务器的最新修改后，必须同时运行SQL数据库的修改脚本，如果忘了跑数据库脚本，那么会出现严重的问题。
传统的解决方案就是在一个固定的文件夹中，将需要跑的SQL脚本放在里面。开发人员在合作的时候，A修改了数据库，在B遇到问题的时候，可能需要交流沟通一下，去跑需要的脚本。在项目上线的过程中，也是运维人员在规定的文件夹中，找到需要跑的SQL脚本。运行它们。<br>
Flyway等migration工具就是要把开发人员和运维人员从以上这些场景的繁琐工作中解放出来，如果使用maven的话，那么在项目编译（SpringBoot运行Application）的时候，SQL数据库的改动就自动进入数据库，只要启动成功，开发或者运维人员对SQL数据库的migrate过程是无感知的，项目依然可以照常运行。

## 0x02 依赖

		<dependency>
		　　<groupId>org.flywaydb</groupId>
		　　<artifactId>flyway-core</artifactId>
		　　<version>4.0.3</version>
		</dependency>

## 0x03 插件

		<plugin>
			<groupId>org.flywaydb</groupId>
			<artifactId>flyway-maven-plugin</artifactId>
			<version>4.0.3</version>
		</plugin>

## 0x04 配置

		flyway:
		  enabled: true
		  baseline-on-migrate: true
		  validate-on-migrate: false
		  
## 0x05 命名
Flyway 默认规约 SQL 脚本文件默认位置是项目的源文件夹下的db/migration 目录。 
SQL 脚本文件及Java 代码类名必须遵循以下命名规则：V[\_][\_\_description] 。
版本号的数字间以小数点（. ）或下划线（_ ）分隔开，版本号与描述间以连续的两个下划线（\_\_ ）分隔开。

## 0x06 版本管理
在mysql对应库内的scheme_version表内存储当前已经执行过的版本，version列存储版本，script存储脚本名称。