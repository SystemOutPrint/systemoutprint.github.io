---
layout: post
title:  "Maven多profile打包"
date:   2017-11-02
author:       "CaiJiahe"
header-img:   "img/tag-bg.jpg"
header-mask:  0.3
catalog:      true
multilingual: true
tags:
    - Maven
---

```xml
	<!-- 通用配置 -->
	<build>
		<resources>
			<resource>
				<directory>${basedir}/src/main/resources</directory>
				<includes>
					<include>**/*</include>
				</includes>
				<filtering>true</filtering>
			</resource>
			<resource>
				<directory>${basedir}/xmls</directory>
			</resource>
		</resources>
	</build>

	<!-- profile对应配置 -->
	<profiles>
		<profile>
			<id>internal</id>
			<activation>
				<activeByDefault>true</activeByDefault>
			</activation>
			<build>
				<resources>
					<resource>
						<directory>${basedir}/src/main/env/internal</directory>
						<includes>
							<include>**/*</include>
						</includes>
						<filtering>true</filtering>
					</resource>
				</resources>
			</build>
		</profile>
		<profile>
			<id>external</id>
			<build>
				<resources>
					<resource>
						<directory>${basedir}/src/main/env/external</directory>
						<includes>
							<include>**/*</include>
						</includes>
						<filtering>true</filtering>
					</resource>
				</resources>
			</build>
		</profile>
	</profiles>
```