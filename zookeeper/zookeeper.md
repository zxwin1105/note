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

    客户端与zkp断开连接后，该节点会被删除。其生命周期与sessionId绑定

4. EPHEMERAL_SEQUENTIAL 临时顺序编号目录节点

    在EPHEMERAL的基础上会给节点添加顺序编号。

5. CONTAINER 容器节点

    3.5.3版本新增，如果container节点下面没有子节点，container节点会被自动清除（清除通过定时任务触发，60s一次）

6. TTL节点 带有过期时间的节点 

    默认是禁用的，需要通过系统配置`zookeeper.extendedTypesEnabled = true`开启

## 2 监听通知机制
