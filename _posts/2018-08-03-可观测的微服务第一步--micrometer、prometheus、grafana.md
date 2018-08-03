---
layout: post
title:  "可观测的微服务第一步--micrometer、prometheus、grafana"
date:   2018-08-01
categories: DevOps
---

## 0x01 micrometer介绍
micrometer一个应用度量的门面框架，自称metrics界里的slf4j，注入各个监控系统的厂商的多维度度量代码在一个中间层，可以自定义监控后端。<br>
在Spring Boot 2.0后，micrometer正式成为了metrics端点的实现，用来对接更多的监控系统。

## 0x02 micrometer使用
在pom.xml中加入
```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-core</artifactId>
</dependency>
```
在application.yml中加入
```yml
management:
  endpoints:
    web:
      exposure:
        include: "*"
  endpoint:
    metrics:
      enabled: true
```
在代码中使用
```java
// counter的name是startMatchSuccess，tag是matchqueue=single
private Counter singleStartSuccessCnt = Metrics.counter("startMatchSuccess", "matchqueue", "single");

// 计数
singleStartSuccessCnt.increment();
```

## 0x03 prometheus配置
修改项目的pom.xml
```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-core</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```
修改项目的application.yml
```yml
management:
  metrics:
    export:
      prometheus:
        enabled: true
  endpoints:
    web:
      exposure:
        include: "*"
  endpoint:
    prometheus:
      enabled: true
    metrics:
      enabled: true
```
启动项目，查看`/actuator/prometheus`下是否有内容。然后在官网上下好prometheus，配置好`prometheus.yml`。
```yml
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']
    
    # 新增的matchqueue job，以5s每次的频率监控端点'http://localhost:8010/actuator/prometheus'
  - job_name: 'matchqueue'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 5s
    static_configs:
    - targets: ['localhost:8010']

```

然后启动

    ./prometheus --config.file=prometheus.yml

在启动界面的query框内输入`name{tag:key=tag:value}`来查看对应的仪表信息。

## 0x04 grafana配置
下好安装包，启动，配置好prometheus之后然后在dashborad里面`Add Query`好`startMatchSuccess_total{matchqueue="single"}`，就能看到仪表显示。