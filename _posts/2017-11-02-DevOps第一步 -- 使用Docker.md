---
layout: post
title:  "DevOps第一步 -- 使用Docker"
date:   2017-11-02
categories: DevOps
---

## 0x01 Docker
Docker 是一个开源的应用容器引擎，让开发者可以打包他们的应用以及依赖包到一个可移植的容器中，然后发布到任何流行的 Linux 机器上，也可以实现虚拟化。容器是完全使用沙箱机制，相互之间不会有任何接口。

## 0x02 Dockerfile
__FROM \<image name\>__：基于哪个Docker镜像来构建新镜像。<br>
__MAINTAINER \<author name\>__：作者和联系方式。<br>
__RUN \<command\>__：在Docker中执行一个bash或者exec。<br>
__ADD \<src\> \<dest\>__：复制文件指令。dest是容器内的路径，src可以是URL或者是启动配置上下文中的一个文件。<br>
__COPY \<src\> \<dest\>__：和ADD类似，但是如果src是归档文件，则不会解压。<br>
__CMD ["executable", "param1", "param2"]__：真正执行命令，docker run后面的参数将覆盖前面命令后面的参数。多条只有最后一条会生效。<br>
__ENTRYPOINT ["executable", "param1", "param2"]__：真正执行命令，docker run后面的参数将追加到命令后面。多条只有最后一条会生效。<br>
__WORKDIR \</path/to/workdir\>__：指定CMD、ENTRYPOINT、RUN命令的工作目录。<br>
__EXPOSE \<port1\> \<port2\>__：指定容器在运行时监听的端口。<br>
__ENV \<key\> \<value\>__：指定环境变量。<br>
__VOLUME ["/data"]__：授权从容器到主机访问的目录。<br>


## 0x03 Docker常用命令
* __docker search docker-image__ 搜索docker镜像。
* __docker pull docker-image__ 下载docker镜像。
* __docker build --tag xxx:tag dir__ 构建镜像。
* __docker run --name xxx docker-image__ 指定容器的名字。
* __docker run -d docker-image__ 后台运行根据镜像运行一个容器。
* __docker run -p hostip:hostport:containerport docker-image__ 端口映射。
* __docker ps__ 查看当前所有运行的docker容器。
* __docker ps -a__ 查看所有运行过的docker容器。
* __docker kill xx__ 杀掉当前运行的docker容器。
* __docker rmi xx__ 删除镜像。
* __docker start xxx__ 启动容器。
* __docker restart xxx__ 重启容器。
* __docker stop xxx__ 停止容器。
* __docker logs__ 查看容器日志。
* __docker run --interactive --tty docker-image__ 创建带有交互式的容器。
* __docker-machine ip default__ 查看当前docker的默认ip。

## 0x04 fgs-env的Dockerfile
Dockerfile demo

		FROM centos
		MAINTAINER caijiahe <caijiahe@ledo.com>

		COPY cmake-3.10.0-rc4/ /export/cmake
		COPY jsoncpp/ /export/jsoncpp/
		COPY rabbitmq-c/ /export/rabbitmq-c/

		RUN yum install make -y
		RUN yum install gcc-c++ -y
		RUN yum install openssl-devel -y

		WORKDIR /export/cmake
		RUN ./configure
		RUN make
		ENV PATH /export/cmake/bin:$PATH

		WORKDIR /export/jsoncpp
		RUN cmake -G "Unix Makefiles"
		RUN make
		RUN make install

		WORKDIR /export/rabbitmq-c
		RUN cmake .
		RUN cmake --build .
		RUN make
		RUN make install
		RUN echo "/usr/local/lib64" >> /etc/ld.so.conf
		RUN ldconfig


## 0x05 扩展阅读

[windows上映射端口无法访问的问题](http://www.wangminli.com/?p=1179)