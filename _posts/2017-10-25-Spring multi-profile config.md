---
layout: post
title:  "Spring multi-profile config"
date:   2017-10-25
categories: Spring
---

## 0x01 yaml

可以通过profile在单个文件里定义多个yaml文件
```yaml
server:
    address: 192.168.1.100
---
spring:
    profiles: development
server:
    address: 127.0.0.1
---
spring:
    profiles: production
server:
    address: 192.168.1.120
```

当spring.profiles.active为development，address使用127.0.0.1。profile为production的时候，address使用192.168.1.120。没有profile的时候，address使用192.168.1.100。<br>
默认的profile是default。所以在yaml文件中，我们可以设置一个security.user.password的值，供default使用。

```yaml
server:
  port: 8000
---
spring:
  profiles: default
security:
  user:
    password: weak
```

下面的这个例子，password总是被各个profile共享的。

```yaml
server:
  port: 8000
security:
  user:
    password: weak
```
Spring profile可以通过'!'来排除，至少有一个肯定的profile必须匹配，并且没有任何否定的profile匹配才能使用该配置。