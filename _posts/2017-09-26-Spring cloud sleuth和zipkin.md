---
layout: post
title:  "Spring cloud sleuth和zipkin"
date:   2017-09-26
categories: Spring
---

## 0x01 介绍
__Sleuth__:为SpringCloud应用实现分布式追踪解决方案，其兼容了Zipkin, HTrace和log-based追踪。<br>
__zipkin__:一个开源的分布式追踪系统。在微服务架构下，它用于帮助收集排查潜在问题的时序数据。它同时管理数据收集和数据查询。


## 0x02 sleuth接入
依赖：

		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-sleuth</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-sleuth-zipkin</artifactId>
		</dependency>

采样器配置：
	
	@Bean
    public AlwaysSampler defaultSampler() {
        return new AlwaysSampler();
    }
	
application.yml配置

		spring:
		  zipkin:
			base-url: http://localhost:9411
		  sleuth:
			sampler:
			  percentage: 1.0
		
## 0x03 zipkin
依赖：

		<dependency>
			<groupId>io.zipkin.java</groupId>
			<artifactId>zipkin-autoconfigure-ui</artifactId>
		</dependency>
		<dependency>
			<groupId>io.zipkin.java</groupId>
			<artifactId>zipkin-server</artifactId>
		</dependency>	
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-sleuth</artifactId>
		</dependency>	
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-sleuth-zipkin</artifactId>
		</dependency>

代码:

		@SpringBootApplication
		@EnableZipkinServer
		public class ZipkinApplication {
			public static void main(String[] args) {
				SpringApplication.run(ZipkinApplication.class, args);
			}
		}