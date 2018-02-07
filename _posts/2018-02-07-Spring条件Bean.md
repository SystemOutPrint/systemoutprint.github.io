---
layout: post
title:  "Spring条件Bean"
date:   2018-02-07
author:       "CaiJiahe"
header-img:   "img/tag-bg.jpg"
header-mask:  0.3
catalog:      true
multilingual: true
tags:
    - Spring
---

## 0x01 SpEL
Spring 3.0开始支持SpEL。
```java
public class MyDao {
  @Value("#{systemProperties['os.arch'].equals('x86') ? winDataSource : unixDataSource}")
  private DataSource datasource;
  ...
}
```

## 0x02 Profile
Spring 3.1开始支持基于Profile的Configuration。
```java
@Configuration
public class MyConfiguration {

  @Bean
  public EmailService emailerService(){
    if (System.getProperty("os.name").contains("Windows")){
      return new WindowsEmailService();
    }
    return new LinuxEmailService();
  }
}
```
或者如下:
```java
@Configuration
@Profile("Linux")
public class LinuxConfiguration {

  @Bean
  public EmailService emailerService() {
    return new LinuxEmailService();
  }
}

@Configuration 
@Profile("Windows") 
public class WindowsConfiguration {

  @Bean
  public EmailService emailerService() {
    return new WindowsEmailService();
  }
}
```

## 0x03 Conditional
Spring 4.0开始支持Conditional
```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Conditional;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MyConfiguration {

  @Bean(name="emailerService")
  @Conditional(WindowsCondition.class)
  public EmailService windowsEmailerService(){
      return new WindowsEmailService();
  }

  @Bean(name="emailerService")
  @Conditional(LinuxCondition.class)
  public EmailService linuxEmailerService(){
    return new LinuxEmailService();
  }
}
```