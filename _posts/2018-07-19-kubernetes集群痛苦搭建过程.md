---
layout: post
title:  "kubernetes 1.11集群痛苦搭建过程"
date:   2018-07-19
categories: kubernetes
---

## 0x01 安装docker
yum install docker -y

可能遇到开启selinux后，docker无法启动的问题。要去`/etc/sysconfig/docker`中将--selinux-enabled设置为false。

## 0x02 下载离线镜像
找一台能翻墙的机器或者去阿里云的镜像仓库去下载所需镜像，从阿里云下载的需要docker tag成原命名镜像。

		k8s.gcr.io/kube-proxy-amd64                v1.11.0             
		k8s.gcr.io/kube-apiserver-amd64            v1.11.0             
		k8s.gcr.io/kube-controller-manager-amd64   v1.11.0            
		k8s.gcr.io/kube-scheduler-amd64            v1.11.0            
		k8s.gcr.io/coredns                         1.1.3              
		gcr.io/kubernetes-helm/tiller              v2.9.1             
		k8s.gcr.io/etcd-amd64                      3.2.18              
		k8s.gcr.io/kubernetes-dashboard-amd64      v1.8.3              
		quay.io/coreos/flannel                     v0.10.0-amd64       
		k8s.gcr.io/pause-amd64                     3.1                 

## 0x03 参数设置

```bash
# 临时关闭swap
# 永久关闭 注释/etc/fstab文件里swap相关的行
swapoff -a

# 开启forward
# Docker从1.13版本开始调整了默认的防火墙规则
# 禁用了iptables filter表中FOWARD链
# 这样会引起Kubernetes集群中跨Node的Pod无法通信

iptables -P FORWARD ACCEPT

# 配置转发相关参数，否则可能会出错
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
vm.swappiness=0
EOF
sysctl --system

# 加载ipvs相关内核模块
# 如果重新开机，需要重新加载
modprobe ip_vs
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
modprobe nf_conntrack_ipv4
lsmod | grep ip_vs
```

## 0x04 使用kubeadm初始化
```bash
kubeadm init --kubernetes-version=v1.11.0 --pod-network-cidr=10.244.0.0/16
```
### network not ready
如果执行`systemctl status kubelet -l`后看到如下输出不要慌。

		Container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized

其实这段输出并没有什么问题，等会安装`flannel`网络插件后，Network就会初始化了。

### failed pulling image "k8s.gcr.io/pause:3.1"
执行下`docker tag k8s.gcr.io/pause-amd64:3.1 k8s.gcr.io/pause:3.1`就好了。

## 0x05 使用kubeadm安装flannel插件
在master上执行

		wget https://raw.githubusercontent.com/coreos/flannel/v0.10.0/Documentation/kube-flannel.yml
		kubectl apply -f kube-flannel.yml 

这个时候等待worker的node加入就可以了。

## 0x06 kubeadm join
在worker的node上执行`kubeadm join 10.237.38.102:6443 --token xxx --discovery-token-ca-cert-hash sha256:xxx`就好了。