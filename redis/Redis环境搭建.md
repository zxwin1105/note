# Redis环境搭建

## 环境：CentOS7 + Docker + Redis

## 1 单机环境搭建

    docker安装单机环境的redis。单机环境适合日常个人学习使用，搭建步骤简单。

### 1.1 拉取redis镜像

```shell
docker pull redis[:version]
```

    从docker默认仓库获取镜像速度比较慢，不稳定。可以将仓库修改为国内镜像。修改 ` /etc/docker/daemon.json ` 文件内容如下。

```vim
{
"registry-mirrors": [
        "http://hub-mirror.c.163.com",
        "https://docker.mirrors.ustc.edu.cn",
        "https://registry.docker-cn.com"
    ]
}
```

    修改之后，重启docker,` systemctl restart docker `

### 1.2 配置启动redis容器

    在获取redis镜像之后，可以直接使用` docker run --name redis -it redis[:version] redis-server `命令创建redis容器，并启动redis服务。但是这种方式不利于对redis配置和数据的修改，需要每次都进入到redis容器中修改。

    可以在创建容器时将redis的配置和数据目录挂载到本机目录。后续可以直接修改本机目录的数据，重启容器即可。

1. 本机创建挂载目录

```shell
mkdir -p /volume/redis/data /volume/redis/conf
```

2. redis配置，在conf目录下生成redis.conf配置。单机基本配置如下

```vim
# 守候进程启动
# 一定设置为no(避坑),如果已守护进程方式启动，docker会认为没有任务需要运行，直接退出
# docker 在启动容器时可以使用 -d 参数进行后台启动。
daemonize no

# 端口
port 6379

# 数据库的个数
databases 16

# 数据备份目录（容器目录）
dir /data

# 开启AOF
appendonly yes
appendfsync everysec

# AOF重写策略
no-appendfsync-on-rewrite yes
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

3. 使用docker run命令创建容器

```shell
docker run --name=dc-redis \
-p 6379:6379 --privileged=true \
-v /docker/volume/redis/data:/data \
-v /docker/volume/redis/conf/redis.conf:/etc/redis/redis.conf \
-itd redis:latest redis-server
```

    使用` docker ps -a` 命令查看是否启动成功。如果发现容器状态不是 UP，可以使用` docker logs -f container-names` 来查看日志。

## 2 主从环境搭建

    主从环境搭建需要多个redis服务。服务环境如下：

| docker容器地址:端口   | 服务类型   |
| --------------- | ------ |
| 172.18.0.2:6380 | master |
| 172.18.0.3:6381 | slave  |
| 172.18.0.4:6382 | slave  |

### 2.1 环境准备

1. 在本机创建redis服务的挂载目录。

```shell
# 创建redis数据挂载目录
mkdir -p /volume/redis/data/masterSlave/6380 \
/volume/redis/data/masterSlave/6381 \
/volume/redis/data/masterSlave/6382

# 创建redis配置挂载目录

mkdir -p /volume/reids/conf/masterSlave
```

2. 配置文件

    master配置文件` vim /volume/reids/conf/masterSlave/redis-6380.conf`

```vim
bind 0.0.0.0
# 守候进程启动
daemonize no

# 端口
port 6380

# pidfile
pidfile /var/run/redis_6380.pid

# 数据库的个数
databases 16

# 数据备份目录
dir /data

# 主从配置 redis5.0之前是哟个命令slaveof 

# 从172.18.0.2 6380获取数据 slaveof节点配置
# 172.18.0.2 查询docker容器的ip
# replicaof 172.18.0.2 6380 

# 开启从节点只读
# replica‐read‐only yes

# 开启AOF
appendonly yes
appendfsync everysec
appendfilename appendonly-6380.aof
# AOF重写策略
no-appendfsync-on-rewrite yes
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

     复制配置到redis-6381.conf和redis-6382.conf并修改port。

### 2.2 网络配置

    搭建redis主从结构是通过docker创建不同角色容器来实现的。redis从实例需要指导主实例的ip，进行主从复制。

    创建docker网络，  <mark> --subnet详情待补充</mark>

```shell
docker network create --subnet=172.18.0.2/16 redis-ms-network
```

### 2.3 启动redis容器

```shell
docker run --name=ms-redis-6380 \
-p 6380:6379 --privileged=true --ip 172.18.0.2 --net redis-ms-network \
-v /tmp/volume/redis/data/masterSlave/6380:/data \
-v /tmp/volume/redis/conf/masterSlave/redis-6380.conf:/etc/redis/redis.conf \
-itd redis:latest redis-server /etc/redis/redis.conf
```

```shell
docker run --name=ms-redis-6381 \
-p 6381:6379  --privileged=true --ip 172.18.0.3 --net redis-ms-network \
-v /tmp/volume/redis/data/masterSlave/6381:/data \
-v /tmp/volume/redis/conf/masterSlave/redis-6381.conf:/etc/redis/redis.conf \
-itd redis:latest redis-server /etc/redis/redis.conf
```

```shell
docker run --name=ms-redis-6382 \
-p 6382:6379 --privileged=true --ip 172.18.0.4 --net redis-ms-network \
-v /tmp/volume/redis/data/masterSlave/6382:/data \
-v /tmp/volume/redis/conf/masterSlave/redis-6382.conf:/etc/redis/redis.conf \
-itd redis:latest redis-server /etc/redis/redis.conf
```

### 2.4 测试主从

    在容器启动成功后，可以先后进入redis主从客户端进行测试。

  ` docker exec -it ms-redis-6380 redis-cli -p 6380`

     使用 redis 的 info命令查看是否配置成功

```shell
# Replication 主节点信息
role:master
connected_slaves:2
slave0:ip=172.18.0.3,port=6381,state=online,offset=1121,lag=0
slave1:ip=172.18.0.4,port=6382,state=online,offset=1121,lag=0
master_failover_state:no-failover
master_replid:d381bc991cd30e6f91ad67d7686661d9237ed0f2
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:1121
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:1121 

# Replication 从节点信息
role:slave
master_host:172.18.0.2
master_port:6380
master_link_status:up
master_last_io_seconds_ago:7
master_sync_in_progress:0
slave_read_repl_offset:211
slave_repl_offset:211
slave_priority:100
slave_read_only:1
replica_announced:1
connected_slaves:0
master_failover_state:no-failover
master_replid:d381bc991cd30e6f91ad67d7686661d9237ed0f2
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:211
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:198
repl_backlog_histlen:14
```

## 3 哨兵模式搭建

    redis中sentinel哨兵也是一个redis实例，不过sentinel不对外提供服务，而是否则对提供服务的redis实例进行监控。

    这里使用2中搭建的注册环境作为提供服务的redis实例。

    sentinel环境如下：

| docker容器地址 | 端口    |
| ---------- | ----- |
| 172.18.0.5 | 16380 |
| 172.18.0.6 | 16381 |
| 172.18.0.7 | 16382 |

### 3.1 配置哨兵

    对哨兵节点进行配置。sentinel-16381.conf，sentinel-16382.conf，sentinel-16383.conf

```vim
# 设置绑定的ip
bind 0.0.0.0

# 设置端口号
port 16380

# 是否守护进程运行（后台运行）
daemonize no

# pidfile
pidfile /var/run/redis_6380.pid

# 配置redis主服务地址 2表示2个或以上sentinel才能确定主节点挂了
sentinel monitor master 172.18.0.2 6380 2
```

### 3.2 启动容器

    由于哨兵实例不会存储业务数据，所以不必挂载/data目录。

    启动容器命令

```shell
docker run --name=sentinel-redis-16380 \
-p 16380:6379 --privileged=true --ip 172.18.0.5 --net redis-ms-network \
-v /tmp/volume/redis/conf/sentinel/sentinel-16380.conf:/etc/redis/redis.conf \
-itd redis:latest redis-sentinel /etc/redis/redis.conf
```

```shell
docker run --name=sentinel-redis-16381 \
-p 16381:6379 --privileged=true --ip 172.18.0.6 --net redis-ms-network \
-v /tmp/volume/redis/conf/sentinel/sentinel-16381.conf:/etc/redis/redis.conf \
-itd redis:latest redis-sentinel /etc/redis/redis.conf
```

```shell
docker run --name=sentinel-redis-16382 \
-p 16382:6379 --privileged=true --ip 172.18.0.7 --net redis-ms-network \
-v /tmp/volume/redis/conf/sentinel/sentinel-16382.conf:/etc/redis/redis.conf \
-itd redis:latest redis-sentinel /etc/redis/redis.conf
```

## 4 集群模式

    Redis集群模式最少需要配置3个小集群，每个小集群是一个主从模式。集群模式环境如下：

| 集群  | 地址    | 端口    |
| --- | ----- | ----- |
| 1   | 宿主机ip | 16390 |
| 2   | 宿主机ip | 16391 |
| 3   | 宿主机ip | 16392 |
| 4   | 宿主机ip | 16393 |
| 5   | 宿主机ip | 16394 |
| 6   | 宿主机ip | 16395 |

配置文件模板 redis-cluster.tmpl

```vim
daemonize no

port ${PORT}

pidfile /var/run/redis_${PORT}.pid

dir /dat

# 集群配置
cluster-enabled yes
cluster-config-file node-${PORT}.conf
cluster-node-timeout 10000
# 集群节点 IP，填写宿主机的 IP
cluster-announce-ip 192.168.56.101
# 集群节点映射端口
cluster-announce-port ${PORT}
# cluster-announce-bus-port ${PORT}

# 关闭保护模式
protected-mode no

appendonly yes

# 设置密码 访问redis的密码
requirepass zxwin

# 集群节点之间，访问的密码
masterauth zxwin
```

创建配置文件 进入配置文件目录执行

```shell
for port in $(seq 16390 16395); do \
    touch redis-${port}.conf &&
    PORT=${port} envsubst <redis-cluster.tmpl> redis-${port}.conf; \
done
```

创建数据存储目录

```shell
for port in $(seq 16390 16395);do \
    mkdir ${port}; \
done
```

创建容器

```shell
for port in $(seq 16390 16395); do \
    docker run --name c-redis-${port} -p ${port}:6379 \
    -itd --privileged=true --net host \
    -v /tmp/volume/redis/conf/cluster/redis-${port}.conf:/etc/redis/redis.conf \
    -v /tmp/volume/redis/data/cluster/${port}:/data \
    redis:7.0.4 redis-server /etc/redis/redis.conf; \
done
```

此时只是创建并启动了6个redis服务，但是他们还没有构成集群，执行一下命令构建集群。

```shell
docker exec -it c-redis-16390 redis-cli -p 16390 \
-a zxwin --cluster create --cluster-replicas 1 \
192.168.56.101:16390 192.168.56.101:16391 192.168.56.101:16392 \
192.168.56.101:16393 192.168.56.101:16394 192.168.56.101:16395
```

一下是输入命令后，redis对集群的分片计划，如果同意输入yes

```shell
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 192.168.56.101:16394 to 192.168.56.101:16390
Adding replica 192.168.56.101:16395 to 192.168.56.101:16391
Adding replica 192.168.56.101:16393 to 192.168.56.101:16392
>>> Trying to optimize slaves allocation for anti-affinity
[WARNING] Some slaves are in the same host as their master
M: 0f2ae064f630040d5fee37ce70066dd636cbfb09 192.168.56.101:16390
   slots:[0-5460] (5461 slots) master
M: 95c2d5c234f07ffa067ea4d82d299b7623a97b55 192.168.56.101:16391
   slots:[5461-10922] (5462 slots) master
M: acbc99a7f50e1b11c65ae3b184e0ece6da315c3c 192.168.56.101:16392
   slots:[10923-16383] (5461 slots) master
S: 55fc546b4b58685ae29129a548afbe4a59905a35 192.168.56.101:16393
   replicates acbc99a7f50e1b11c65ae3b184e0ece6da315c3c
S: 86821c772434c878f1ed91b414a2f3f19c2458a0 192.168.56.101:16394
   replicates 0f2ae064f630040d5fee37ce70066dd636cbfb09
S: 862feb247e9b820927d909daf9493c8e754ba634 192.168.56.101:16395
   replicates 95c2d5c234f07ffa067ea4d82d299b7623a97b55
Can I set the above configuration? (type 'yes' to accept): yes
```

进入集群

```shell
docker exec -it c-redis-16390 redis-cli -p 16390 \
-a zxwin -c
```

查看集群信息 cluster nodes
