---
layout: post
title:  "DevOps第二步 -- 用Docker compose来管理微服务容器"
date:   2017-11-03
author:       "CaiJiahe"
header-img:   "img/tag-bg.jpg"
header-mask:  0.3
catalog:      true
multilingual: true
categories: DevOps
---

## 0x01 Docker compose
Docker Compose是一个用来定义和运行复杂应用的Docker工具。使用Compose可以在一个文件中定义一个多容器应用，然后使用一条命令来启动应用，完成一切准备工作。

## 0x02 compose配置文件
附上demo
```yml
version: "3"
services:
  base:
    build: base
  eureka:
    build: ../kof-eureka-server
    ports:
      - "8761:8761"
    depends_on:
      - "base"
  matchqueue:
    build: ../kof-matchqueue
    ports:
      - "8000:8000"
    depends_on:
      - "base"
  room:
    build: ../kof-room
    ports:
      - "8004:8004"
    depends_on:
      - "base"
  redis:
    build: redis
    ports:
      - "6379:6379"
  rabbitmq:
    image: "rabbitmq:latest"
    ports:
      - "5672:5672"
```
这个配置文件声明了base、eureka、matchqueue、room、redis、rabbitmq这些服务，其中room、matchqueue、eureka依赖base。rabbitmq直接基于官方提供的镜像。


## 0x03 Docker compose元素详解
#### build
```yml
services:
  service:
    build:
      context: ./dir # 包含Dockerfile目录名或指向git仓库的一个url
      dockerfile: Dockerfile-alternate # Dockerfile名
      args:
        buildno: 1 # Dockerfile的build的参数，可以是map/list(yml格式)。
```
__cache_from__：v3.2之后支持，一个镜像的列表，可能会在build的过程中当做cache使用。<br>
__labels__：v3.3之后支持，docker镜像的标签。

#### image
```yml
services:
  service:
    image: redis:latest  # 基于redis:latest镜像
```

#### cap_add/cap_drop
添加或放弃容器的Linux能力(swarm模式会忽略这个参数)。
```yml
cap_add:
  - ALL
cap_drop:
  - NET_ADMIN
  - SYS_ADMIN
```

#### depend_on
该服务依赖什么服务。

#### command
覆盖Dockerfile里的CMD。
```yml
command: bundle exec thin -p 3000 # 类似Dockerfile中的 CMD ["bundle", "exec", "thin", "-p", "3000"]
```

#### configs
授予一个服务访问某个外部配置文件的权限。<br>

__短语法：__<br>
```yml
version: "3.3"
services:
  redis:
    image: redis:latest
    deploy:
      replicas: 1
    configs: # 只有redis可以访问这两个配置
      - my_config
      - my_other_config
configs: # 声明my_config、my_other_config配置
  my_config:
    file: ./my_config.txt
  my_other_config:
    external: true
```

__长语法：__<br>
```yml
version: "3.3"
services:
  redis:
    image: redis:latest
    deploy:
      replicas: 1
    configs:
      - source: my_config
        target: /redis_config
        uid: '103'
        gid: '103'
        mode: 0440
configs:
  my_config:
    file: ./my_config.txt
  my_other_config:
    external: true
```

#### environment
设置环境变量。
```yml
environment:
  - RACK_ENV=development
  - SESSION_SECRET
```
#### env_file
从文件中获取环境变量，可以为单独的文件路径或列表。
如果通过 docker-compose -f FILE 指定了模板文件，则env_file中路径会基于模板文件路径。
如果有变量名称与 environment 指令冲突，则以后者为准。
```yml
env_file: .env
env_file:
  - ./common.env
  - ./apps/web.env
  - /opt/secrets.env
```

#### net
设置网络模式。
```yml
net: "bridge"
net: "none"
net: "container:[name or id]"
net: "host"
```

#### cgroup_parent
可选参数，一个Container的父cgroup(swarm模式会忽略这个参数)。

#### container_name
容器名，因为容器名必须要唯一，所以如果赋值后，就不能水平扩展了。

#### deploy
只在使用docker stack deploy部署在swarm环境中有效。在docker-compose up/docker-compose run中自动失效。

#### volumes
卷挂载路径设置。可以设置宿主机路径 （HOST:CONTAINER） 或加上访问模式 （HOST:CONTAINER:ro）。

#### healthcheck
可以根据特定条件做指定的操作，其他容器可以根据某些条件判断来阻塞等待。<br>
demo:
		
	# docker-compose.yml
	version: "2.1"
	services:
      ...
	  a:
        depends_on:
          configserver:
            condition: service_healthy
      ...
	 
	 
	# configserver的Dockerfile
	
	...
	HEALTHCHECK --interval=10s --timeout=3s --retries=30 
	             CMD curl -f http://localhost:10888/health || exit 1
	...

configserver的Dockerfile中的HEALTHCHECK表明每次间隔10秒一共做30次轮询，每次超时时间是3秒，条件是url返回内容。

#### dns
配置 DNS 服务器。可以是一个值，也可以是一个列表(swarm模式会忽略这个参数)。
```yml
dns: 8.8.8.8
dns:
  - 8.8.8.8
  - 9.9.9.9
```

#### dns_search
配置 DNS 搜索域。可以是一个值，也可以是一个列表(swarm模式会忽略这个参数)。
```yml
dns_search: example.com
dns_search:
  - domain1.example.com
  - domain2.example.com
```

#### deploy
##### mode
deploy提供了一个模式选项，它的值有global和replicated两个，默认是replicated模式。<br>
这两个模式的区别是：<br>
global：每个集群每个服务实例启动一个容器，就像以前启动 Service 时一样。<br>
replicated：用户可以指定集群中实例的副本数量。<br>

##### replicas
指定副本数量

```yml
deploy:
  replicas: 6
```

##### update_config
Swarm 就提供了一个升级回滚的功能。当服务升级出现故障时，超过重试次数则停止升级的功能，这也很方便，避免让错误的应用替代现有正常服务。<br>
这个选项用于告诉 Compose 使用怎样的方式升级，以及升级失败后怎样回滚原来的服务。<br>
* parallelism: 服务中多个容器同时更新。
* delay: 设置每组容器更新之间的延迟时间。
* failure_action: 设置更新失败时的动作，可选值有 continue 与 pause (默认是：pause)。
* monitor: 每次任务更新失败后监视故障的持续时间 (ns|us|ms|s|m|h) (默认：0s)。
* max_failure_ratio: 更新期间容忍的故障率。

##### resources
替代了之前的类似cpu_shares, cpu_quota, cpuset, mem_limit, memswap_limit这种选项。
```yml
resources:
  limits:
    cpus: '0.001'
    memory: 50M
  reservations:
    cpus: '0.0001'
    memory: 20M
```

##### restart_policy
设置如何重启容器，毕竟有时候容器会意外退出。<br>
* condition：设置重启策略的条件，可选值有none, on-failure和any (默认：any)。
* delay：在重新启动尝试之间等待多长时间，指定为持续时间（默认值：0）。
* max_attempts：设置最大的重启尝试次数，默认是永不放弃。
* window：在决定重新启动是否成功之前要等待多长时间，默认是立刻判断，有些容器启动时间比较长，指定一个“窗口期”非常重要。

