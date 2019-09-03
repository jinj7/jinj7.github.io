---
layout: post
title: Linux network namespace
categories: [linux, network]
keywords: network namespace
---


### 1、创建网桥

```
# 新建网桥br0
brctl addbr br0

# 关闭STP（生成树协议）
brctl stp br0 off

# 为网桥设置IP地址
ifconfig br0 192.168.10.1/24 up
```

### 2、创建network namespace

```
# 创建network namespace ns1
ip netns add ns1
```

### 3、创建veth pair，用来连接ns1和default namespace

```
# 创建veth pair
ip link add veth-ns1 type veth peer name br0.1

# 把veth-ns1放进namespace ns1中
ip link set veth-ns1 netns ns1
```

veth pair的两端分别是veth-ns1和br0.1

### 4、配置namespace ns1内部的网络

```
# 启动一个属于ns1的shell
ip netns exec ns1 sh

# 查看当前的端口
ip link show

# veth-ns1改名为eth0
ip link set dev veth-ns1 name eth0

# 给eth0分配一个IP并up
ifconfig eth0 192.164.10.11/24 up

# lo up
ip link set dev lo up

# 查看IP地址配置
ip addr show

# 设置默认路由
ip route add default via 192.168.10.1 dev eth0
```

### 5、配置default namespace

```
# br0.1 up
ifconfig br0.1 up

# 把br0.1添加到网桥br0上
brctl addif br0 br0.1
```

现在已经可以ping通namespace ns1中的IP地址了。  

如果还想让namespace ns1访问外网，还需要做如下配置：
```
# 开启端口转发
echo 1 > /proc/sys/net/ipv4/ip_forward

# 添加SNAT规则
iptables -t nat -A POSTROUTING -s 192.168.10.0/24 -o eth0 -j MASQUERADE
```
