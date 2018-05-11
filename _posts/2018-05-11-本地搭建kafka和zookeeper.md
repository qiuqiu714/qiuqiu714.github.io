---
title: 本地搭建kafka和zookeeper
date: 2018-05-11 15:00:00
categories:
- java
tags: kafka, zookeeper
---

因为工作需要借助kafak和spark读取日志进行处理，上一次记录了搭建spark和hdfs的过程：
[本地搭建hdfs以及spark环境](https://bosszz.github.io/java/2018/05/09/%E6%9C%AC%E5%9C%B0%E6%90%AD%E5%BB%BAhdfs%E4%BB%A5%E5%8F%8Aspark%E7%8E%AF%E5%A2%83/)

在这一篇，将记录一下搭建kafka和zookeeper的过程

# 一、zookeeper搭建
因为对于kafka的启动zookeeper是必须的，所以首先搭建zookeeper

## 1.准备
从zookeeper官网下载tar包

[http://zookeeper.apache.org/releases.html#download](http://zookeeper.apache.org/releases.html#download)

直接下载最新版

这里搭建的是本地模拟集群环境，所以新建三个文件夹
```
$ mkdir /Users/zhoujy/zookeeper/zookeeper-1
$ mkdir /Users/zhoujy/zookeeper/zookeeper-2
$ mkdir /Users/zhoujy/zookeeper/zookeeper-3
```

之后分别解压到三个文件夹

## 2.配置
为每个目录的conf目录下创建zoo.cfg文件

对于zookeeper-1
添加如下内容
``` properties
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/Users/zhoujy/hadoop/hadoop/tmp/zk1/data
dataLogDir=/Users/zhoujy/hadoop/hadoop/tmp/zk1/log
clientPort=2181
server.1=localhost:2287:3387
server.2=localhost:2288:3388
server.3=localhost:2289:3389
```

对于zookeeper-2
添加如下内容
``` properties
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/Users/zhoujy/hadoop/hadoop/tmp/zk2/data
dataLogDir=/Users/zhoujy/hadoop/hadoop/tmp/zk2/log
clientPort=2182
server.1=localhost:2287:3387
server.2=localhost:2288:3388
server.3=localhost:2289:3389
```

对于zookeeper-3
添加如下内容
``` properties
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/Users/zhoujy/hadoop/hadoop/tmp/zk3/data
dataLogDir=/Users/zhoujy/hadoop/hadoop/tmp/zk3/log
clientPort=2183
server.1=localhost:2287:3387
server.2=localhost:2288:3388
server.3=localhost:2289:3389
```

因为是模拟集群，所以端口号互相错开

之后再配置的每个data目录下添加myid文件，内容对应server.x中的x

```
/Users/zhoujy/hadoop/hadoop/tmp/zk1/data/myid 中的内容为1，对应server.1的1
/Users/zhoujy/hadoop/hadoop/tmp/zk2/data/myid 中的内容为2，对应server.2的2
/Users/zhoujy/hadoop/hadoop/tmp/zk3/data/myid 中的内容为3，对应server.3的3
```

## 3.启动

分别运行bin下面的zkServer.sh脚本

```
$ sh bin/zkServer.sh start
```

然后可以使用
```
$ sh bin/zkCli.sh -server localhost:2181
```
来进行连接测试

成功后应该会得到类似的提示
> [zk: localhost:2181(CONNECTED) 0]  


# 二、kafka搭建
搭建完zookeeper就可以搭建kafka了

## 1.准备

从kafka官网下载tar包

[http://kafka.apache.org/downloads](http://kafka.apache.org/downloads)

这里使用的是0.11.0.2的版本

下载后解压，进入kafka目录

## 2.配置

修改config目录下的server.properties配置文件

```
$ vim server.properties
```

添加或修改以下内容：
> listeners=PLAINTEXT://localhost:9092

> zookeeper.connect=localhost:2181

保存

修改config目录下的zookeeper.properties配置文件
```
$ vim zookeeper.properties
```

添加以下内容：
> dataDir=/Users/zhoujy/kafka_2.11-0.11.0.2/tmp/zookeeper

> clientPort=2181

保存

## 3.启动

通过以下命令启动：
```
$ sh bin/kafka-server-start.sh config/server.properties
```

成功启动kafak

## 4.一些常用命令
### (1) 创建一个topic
```
$ sh kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic topic-1
```

### (2) 查看所有topic
```
$ sh kafka-topics.sh --list --zookeeper localhost:2181
```

### (3) 启动生产者客户端
```
$ sh kafka-console-producer.sh --broker-list localhost:9092 --topic topic-1
```
可以往该topic生产数据

### (4) 查看topic中的日志
```
$ sh kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic topic-1 --from-beginning
```

### (5) 查看group的状态
```
$ sh sh kafka-consumer-offset-checker.sh --topic topic-1 --group group-1 --zookeeper localhost:2181
```

# 结尾
至此本地搭建zookeeper和kafka完成，可以在本地spark streaming程序中读取kafka中的数据来进行本地调试了

# 参考资料
- [在本地模拟搭建zookeeper集群环境实例](https://www.cnblogs.com/baihaojie/p/6688358.html)
- [kafka本地单机安装部署](https://blog.csdn.net/w12345_ww/article/details/51952930)
- [kafka 0.10.2 快速入门（译）](https://blog.csdn.net/gk_kk/article/details/68924835)