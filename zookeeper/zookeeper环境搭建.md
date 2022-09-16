# zookeeper环境搭建

    基于Docker搭建zookeeper环境过程。首先需要pull zookeeper镜像。

## 1 单机环境搭建

    zookeeper配置文件

```vim
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
dataDir=/data
# the port at which the clients will connect
clientPort=2181
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1

## Metrics Providers
#
# https://prometheus.io Metrics Exporter
#metricsProvider.className=org.apache.zookeeper.metrics.prometheus.PrometheusMetricsProvider
#metricsProvider.httpPort=7000
#metricsProvider.exportJvmInfo=true
```

    创建并启动zookeeper容器

```shell
docker run --name dc-zk -it -p 2181:2181 \
-v /tmp/volume/zookeeper/conf/zoo.cfg:/conf/zoo.cfg \
-v /tmp/volume/zookeeper/data:/data \
zookeeper:3.7 zkServer.sh start /conf/zoo.cfg
```

    使用客户端连接zk服务

```shell
docker exec -it dc-zk zkCli.sh
```
