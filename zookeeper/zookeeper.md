# zookeeper

    zookeeper是一个用于协调分布式服务的框架，是一个用于存储少量数据基于内存的数据库，主要有两个核心概念：文件系统数据结构+监听通知机制

## 1 文件系统数据结构

    zookeeper的文件系统是一种树形目录结构，每个子目录项被称作znode(目录节点)，和文件系统类型，可以对其节点进行增加、删除。

    zookeeper有一下6中不同功能的目录节点：

1. PERSISTENT 持久化目录节点

    客户端与zkp断开连接口，该节点依旧存在，只要不手动删除，永久存在

2. PERSISTENT_SEQUENTIAL 持久化顺序编号目录节点

    该类型比起PERSISTENT会对节点进行顺序编号

3. EPHEMERAL 临时目录节点

    客户端与zkp断开连接后，该节点会被删除。其生命周期与sessionId绑定。临时节点不能有子节点

4. EPHEMERAL_SEQUENTIAL 临时顺序编号目录节点

    在EPHEMERAL的基础上会给节点添加顺序编号。

5. CONTAINER 容器节点

    3.5.3版本新增，如果container节点下面没有子节点，container节点会被自动清除（清除通过定时任务触发，60s一次）

6. TTL节点 带有过期时间的节点 

    默认是禁用的，需要通过系统配置`zookeeper.extendedTypesEnabled = true`开启

## 2 监听通知机制

    客户端可以注册监听它关注的任意节点、目录节点及递归子目录节点。

1. 监听某个节点，该节点被删除、或者修改时客户端会被通知

2. 监听某个目录，该目录节点创建子节点或者有子节点被删除时客户端会被通知

3. 监听某个目录的递归子节点，则这个目录下面的任意子节点有目录结构的变化，子节点被创建或者删除，或者根节点有数据变化时，对应的客户端将被通知

> 所有的通知都是一次性的，一旦触发，对应的监听会被移除

### 2.1 watch事件类型

    zk中watch事件类型定义在枚举EventType中，如下：

| 枚举属性                   | 描述                        |
| ---------------------- | ------------------------- |
| Node(-1)               | 无                         |
| NodeCreated(1)         | Watcher监听的数据节点被创建时        |
| NodeDeleted(2)         | Watcher监听的数据节点被删除时        |
| NodeDataChanged(3)     | Watcher监听的数据节点内容发生变更时     |
| NodeChildrenChanged(4) | Watcher监听的数据节点的子节点列表发生变更时 |

## 3 acl权限控制

    zookeeper的ACL（Access Control List）权限控制在生产环境很重要。ACL权限可以针对节点设置相关读写等权限，保障数据安全性。

    acl相关的命令：

- getAcl命令：获取某个节点的acl权限信息

- setAcl命令：设置某个节点的acl权限信息

- addauth命令：输入认证授权信息

### 3.1 acl构成

    zk的acl权限主要由3部分组成：权限模式（scheme）：授权对象（ID）：权限信息（permissions）。

1. scheme：代表采用的某种权限机制，包括：
   
   - world：只有一个唯一的ID，`anyone` 代表所有人
   
   - auth：不适用任何ID，代表任何已认证的用户
   
   - digest：使用`username:password` 
   
   - ip：使用客户端的主机IP作为ID
   
   - super

2. id：代表允许访问的用户

3. permissions：权限组合字符串，由crdwa组成，创建权限（create）、删除权限（delete）、读权限（read）、写权限（write）、管理权限（admin）

## 4 zookeeper事务日志

    zookeeper会将每一次客户端的事务操作记录到事务日志中。

> 事务日志的地址可以通过zoo.cfg进行配置 `dataLogDir` ，如果没有配置该项，默认会存储在`dataDir`(必填项)



## 4 环境搭建

```shell
docker run --name dc-zk -it -p 2181:2181 \
-v /tmp/volume/zookeeper/conf/zoo.cfg:/conf/zoo.cfg \
-v /tmp/volume/zookeeper/data:/data \
zookeeper:3.7 zkServer.sh start /conf/zoo.cfg
```
