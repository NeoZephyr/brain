## 环境搭建

### LVS

dip: 192.168.1.151
vip: 192.168.1.150

关闭网络配置管理器

```bash
systemctl stop NetworkManager
systemctl disable NetworkManager
```

创建子接口

```bash
cd /etc/sysconfig/network-scripts/
cp ifcfg-ens33 ifcfg-ens33:1
```

```
BOOTPROTO="static"
DEVICE="ens33:1"
ONBOOT="yes"
IPADDR=192.168.1.150
NETMASK=255.255.255.0
```

重启网络

```bash
service network restart
```

```bash
yum install ipvsadm
ipvsadm -Ln
```

### nginx1

rip: 192.168.1.171
vip: 192.168.1.150

```bash
cp ifcfg-lo ifcfg-lo:1
```

```
DEVICE=lo:1
ONBOOT="yes"
NAME=loopback
IPADDR=192.168.1.150
NETMASK=255.255.255.255
NETWORK=127.0.0.0
BROADCAST=127.255.255.255
```

刷新

```bash
ifup io
```

### nginx2

rip: 192.168.1.172
vip: 192.168.1.150


## ARP

### arp-ignore

0: 只要本机配置了 ip，就能响应请求
1: 请求的目标地址到达对应的网络接口，才会响应请求

### arp-announce

0: 本机上任何网路接口都向外通告，所有网卡都能接受到通告
1: 尽可能避免本网卡与不匹配的目标进行通告
2: 只在本网卡通告

```bash
vi /etc/sysctl.conf
```

```
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.default.arp_ignore = 1
net.ipv4.conf.lo.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
net.ipv4.conf.default.arp_announce = 2
net.ipv4.conf.lo.arp_announce = 2
```

```bash
sysctl -p
```

## ipvsadm

增加一个网关，用于接收数据报文，当有请求到本机后，会交给 lo 去处理

```bash
route add -host 192.168.1.150 dev lo:1
```

查看

```bash
route -n
```

开机启动

```bash
echo "route add -host 192.168.1.150 dev lo:1" >> /etc/rc.local
```

创建 LVS 节点

-A 表示添加集群
-t 表示 tcp 协议
-s 表示负载均衡算法
-p 表示连接持久化时间

```bash
ipvsadm -A -t 192.168.1.150:80 -s rr -p 5
```

添加 2 台服务器

-a 表示添加真实服务器
-r 表示真实服务器地址
-g 表示 DR 模式

```bash
ipvsadm -a -t 192.168.1.150:80 -r 192.168.1.171:80 -g
ipvsadm -a -t 192.168.1.150:80 -r 192.168.1.172:80 -g
```

编辑修改

```bash
ipvsadm -E -t 192.168.1.150:80 -s rr -p 5
```

保存到规则库，否则重启失效

```bash
ipvsadm -S
```

## 查看集群列表

```bash
ipvsadm -Ln
```

## 查看集群状态

```bash
ipvsadm -Ln --stats
```

## 重启 ipvsadm

```bash
service ipvsadm restart
```