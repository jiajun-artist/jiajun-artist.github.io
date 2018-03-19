---
layout: post
title: "Dynomite 应用及运维实例"
tags: [ops]
comments: true
date: 2018-03-19 15:48:29+08:00
---


#### 大致介绍:
```
1.大规模运用于aws和netflix
2.可以跨数据中心进行redis常见操作的同步  
3.dynomite本身类似与redis上的代理, 转发redis操作协议到其他server   
```

#### 项目github地址
```
https://github.com/Netflix/dynomite
```
#### 配置文件参考地址
```
https://github.com/Netflix/dynomite/tree/dev/conf
```
#### 结构原理图
![](https://github.com/Netflix/dynomite/wiki/images/dynomite-architecture.png)
```
https://github.com/Netflix/dynomite/wiki/Architecture
```


部署:
```
yum install -y git autoconf automake libtool openssl-devel net-tools
mkdir /opt/src
cd /opt/src
git clone https://github.com/Netflix/dynomite.git
cd dynomite
autoreconf -fvi
./configure --enable-debug=log
make
src/dynomite -h

mkdir -p /opt/dynomite/bin /opt/dynomite/conf /opt/dynomite/log
cp /opt/src/dynomite/src/dynomite /opt/dynomite/bin

cd /opt/dynomite/conf

复制3个文件
dynomite.pem
recon_iv.pem
recon_key.pem
具体内容去250上查看并复制

跨数据中心则在主上创建文件
dc1.yml
```
#### 配置示例 dc1.yml (2.2.2.2)
```
dyn_o_mite:
  datacenter: dc1
  rack: rack1
  dyn_listen: 0.0.0.0:8101
  dyn_seeds:
  - 1.1.1.1:8101:rack2:dc2:1383429731
  listen: 127.0.0.1:8102
  servers:
  - 127.0.0.1:6390:1
  tokens: '12345678'
  secure_server_option: datacenter
  pem_key_file: /opt/dynomite/conf/dynomite.pem
  data_store: 0
  stats_listen: 0.0.0.0:22221
  datastore_connections: 3
  local_peer_connections: 3
  remote_peer_connections: 3
```

```
在第二个数据中心的机器上做同样操作
创建文件
dc2.yml
```

#### 配置示例 dc2.yml (1.1.1.1)
```
dyn_o_mite:
  datacenter: dc2
  rack: rack2
  dyn_listen: 0.0.0.0:8101
  dyn_seeds:
  - 2.2.2.2:8101:rack1:dc1:12345678
  listen: 127.0.0.1:8102
  servers:
  - 127.0.0.1:6379:1
  tokens: '1383429731'
  secure_server_option: datacenter
  pem_key_file: /opt/dynomite/conf/dynomite.pem
  data_store: 0
  stats_listen: 0.0.0.0:22222
  datastore_connections: 3
  local_peer_connections: 3
  remote_peer_connections: 3
```


```
启动两个数据中心的节点上的dynomite服务
/opt/dynomite/bin/dynomite -c /opt/dynomite/conf/dc1 -o /opt/dynomite/log/dc1.log -d

/opt/dynomite/bin/dynomite -c /opt/dynomite/conf/dc2 -o /opt/dynomite/log/dc2.log -d
```

#### 测试
```
在dc1上进行set del等, 即同步
在dc2上 redis-cli -p 8102 查看同步情况
```

#### 使用
```
需要数据中心间远程同步的操作, 都走dynomite代理的端口, 本机的数据操作则用原始6379端口即可, 对远程机器无影响
```

#### 监控
```
定期的同步监控字段(如version), 值可为时间戳, 监控服务定期检测此字段是否超时或者版本不对
```