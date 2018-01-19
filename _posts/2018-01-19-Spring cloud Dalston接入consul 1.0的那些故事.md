---
layout: post
title:  "Spring cloud Dalston接入consul 1.0的那些故事"
date:   2018-01-19
categories: Spring
---

## 0x01 consul简介
一个服务管理中心。
* 支持多数据中心下，分布式高可用的，服务发现和配置共享。
* consul支持健康检查，允许存储键值对。
* 一致性协议采用 Raft 算法,用来保证服务的高可用。
* 成员管理和消息广播 采用GOSSIP协议，支持ACL访问控制。

## 0x02 版本兼容性
先贴一个github上Spring cloud的issue:<br>
[spring-cloud/spring-cloud-consul Compatibility with Consul 1.0.0 #365](https://github.com/spring-cloud/spring-cloud-consul/issues/365)<br>
问题中描述到的 _java.lang.IllegalArgumentException: Value must not be null_ 也是我所遇到的问题。
这个问题是因为consul 1.0.0将agent检查和服务注销由get修改为put，导致consul对服务的健康检查不通过，所以服务注册到consul上一直是不健康的状态。
Spring cloud Dalston中依赖的Spring cloud consul 1.2.1版本不兼容consul 1.0的新改动，在Spring cloud consul 1.3.0版本已经解兼容了这个问题。<br>
所以把依赖版本升到 _Spring cloud consul 1.3.0 _ 就解决了这个问题。

## 0x03 在consul上实现类似eureka的metadata存储
[spring-cloud-consul/health check Metadata and Consul tags](https://github.com/spring-cloud/spring-cloud-consul/blob/master/docs/src/main/asciidoc/spring-cloud-consul.adoc)<br>
可以使用consul的tags来实现类似eureka的metadata存储。