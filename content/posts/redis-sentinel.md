---
title: "Redis Sentinel"
description: "Redis Sentinel"
tags: ["redis sentinel", "tech"]
category: ["translation"]
date: 2020-04-01T15:47:14+08:00
draft: false
---

# Redis Sentinel Documentation  

[original](https://redis.io/topics/sentinel)

Redis Sentinel provides high availability for Redis. In practical terms this means that using Sentinel you can create a Redis deployment that resists without human intervention certain kinds of failures.

Redis Sentinel 为 Redis提供了高可用性. 实际上，这里的含义是，使用Sentinel你可以创建一个Redis部署，这个Redis部署不用人为介入就可以对抗某些类型的故障.

Redis Sentinel also provides other collateral tasks such as monitoring, notifications and acts as a configuration provider for clients. 

Redis Sentinel 还提供了其他附属的任务，比如，监控，通知还有为clients扮演配置provider.  

This is the full list of Sentinel capabilities at a macroscopical level (i.e. the big picture): 

这是宏观层面上Sentinel的所有功能，也就是big picture: 

* **Monitoring**. Sentinel constantly checks if your master and replica instances are working as expected.  
监控. Sentinel不断地检查是否master和replica实例，是不是按照预期工作.

* **Notification**. Sentinel can notify the system administrator, or other computer programs, via an API, that something is wrong with one of the monitored Redis instances.  
通知. Sentinel可以通知系统管理员，或者其他的程序通过API，哪个Redis实例出错了.

* **Automatic failover**. If a master is not working as expected, Sentinel can start a failover process where a replica is promoted to master, the other additional replicas are reconfigured to use the new master, and the applications using the Redis server are informed about the new address to use when connecting.  
如果一个master实例没有按照预期运行，Sentinel可以开启一个容错进程，在这个进程一个replica实例被晋升为master，其他的额外的replica实例被重新配置来使用新的master，并且使用这个Redis服务器的应用也被通知使用这个新的master地址.  
    
* **Configuration provider**. Sentinel acts as a source of authority for clients service discovery: clients connect to Sentinels in order to ask for the address of the current Redis master responsible for a given service. If a failover occurs, Sentinels will report the new address.  
配置提供者. Sentinel为clients服务发现扮演着一个源认证机构: clients连接到Sentinels是为了根据一个给定的service，获取当前负责的Redis master实例的地址，如果出现了容错，Sentinels会给clients发一个新的地址.  

## Distributed nature of Sentinel  Sentinel的分布式本质

Redis Sentinel is a distributed system:  
Redis Sentinel是一个分布式系统: 

Sentinel itself is designed to run in a configuration where there are multiple Sentinel processes cooperating together. The advantage of having multiple Sentinel processes cooperating are the following:  
Sentinel自己被设计成在一个配置中运行，在这里多个Sentinel进程一起协作. 多个Sentinel进程相互协作的优势如下:  

1. Failure detection is performed when multiple Sentinels agree about the fact a given master is no longer available. This lowers the probability of false positives.  
错误探测执行是在，当多个Sentinel都同意一个给定的master不可用了. 这个机制降低了误报/假阳性的可能性. 

2. Sentinel works even if not all the Sentinel processes are working, making the system robust against failures. There is no fun in having a failover system which is itself a single point of failure, after all.  
即使并非所有的Sentinel都工作的情况下，Sentinel也可以工作，使得系统对抗错误更健壮. 毕竟，一个容错系统他自己是单点故障一点也不好玩. 

The sum of Sentinels, Redis instances (masters and replicas) and clients connecting to Sentinel and Redis, are also a larger distributed system with specific properties. In this document concepts will be introduced gradually starting from basic information needed in order to understand the basic properties of Sentinel, to more complex information (that are optional) in order to understand how exactly Sentinel works.  

Sentinels，Redis实例还有连接在Sentinel和Redis的clients合起来是一个具有特定属性大的分布式系统，在这篇文档中，概念将会被逐渐的介绍，从基本的需要用来理解Sentinel基本属性的信息到更复杂的（可选的）用来理解Sentinel具体是怎样工作信息.  

## Quick Start  
### Obtaining Sentinel 获取Sentinel

The current version of Sentinel is called **Sentinel 2**. It is a rewrite of the initial Sentinel implementation using stronger and simpler-to-predict algorithms (that are explained in this documentation).  

当前的Sentinel版本是Sentinel 2. 这个版本是一个最初Sentinel实现的重写，使用了更强的简单预测算法（这篇文档会阐释）.

A stable release of Redis Sentinel is shipped since Redis 2.8.  

稳定的发布版本是从Redis 2.8开始的.  

New developments are performed in the unstable branch, and new features sometimes are back ported into the latest stable branch as soon as they are considered to be stable.  

新的开发在不稳定分支执行，并且新的功能有时向后移植到最新的稳定分支只要它们被认定为稳定的.  

Redis Sentinel version 1, shipped with Redis 2.6, is deprecated and should not be used.  

Redis Sentinel version 1 搭配 Redis 2.6一起发布，已经被弃用了，不应该被使用.  

### Running Sentinel  

If you are using the redis-sentinel executable (or if you have a symbolic link with that name to the redis-server executable) you can run Sentinel with the following command line:  

如果你在使用redis-sentinel可以执行文件（或者如果你有一个软链接到redis-server的可执行文件）你可以通过一下命令运行Sentinel:  

    redis-sentinel /path/to/sentinel.conf 

Otherwise you can use directly the redis-server executable starting it in Sentinel mode:  

另外你可以直接使用redis-server可执行文件，以Sentinel模式启动它:  

    redis-server /path/to/sentinel.conf --sentinel 

Both ways work the same.   
两种方法一样.

However it is mandatory to use a configuration file when running Sentinel, as this file will be used by the system in order to save the current state that will be reloaded in case of restarts. Sentinel will simply refuse to start if no configuration file is given or if the configuration file path is not writable.  

然而，使用配置文件运行Sentinel是强制的, 因为这个被系统使用的文件要用来保存当前的状态，当前的状态将会被重新加载当重启的时候. Sentinel将会简单的拒绝启动，如果没有给定配置文件，或者配置文件路径不可写入.  

Sentinels by default run listening for connections to TCP port 26379, so for Sentinels to work, port 26379 of your servers must be open to receive connections from the IP addresses of the other Sentinel instances. Otherwise Sentinels can't talk and can't agree about what to do, so failover will never be performed.  

Sentinel默认监听26379端口，所有为了Sentinel能工作，26379端口必须开放着来接收其他Sentinel实例IP的连接. 否则，Sentinel不能交流，不能同意做什么，所以容错将不会被执行.  

### Fundamental things to know about Sentinel before deploying  
### 部署之前需要知道的关于Sentinel的基础的东西

1. You need at least three Sentinel instances for a robust deployment. 
The three Sentinel instances should be placed into computers or virtual machines that are believed to fail in an independent way. So for example different physical servers or Virtual Machines executed on different availability zones.  
你至少需要三个Sentinel实例，来实现一个健壮的部署. 三个Sentinel实例应该被放在三台不同的物理机或者虚拟机，在三个不同的可用地区执行.  

2. Sentinel + Redis distributed system does not guarantee that acknowledged writes are retained during failures, since Redis uses asynchronous replication. However there are ways to deploy Sentinel that make the window to lose writes limited to certain moments, while there are other less secure ways to deploy it.  
Sentinel + Redis分布式系统不会保证接收的写入会被保留在出现错误的时候，既然Redis使用异步复制. 然而，这里有方法来部署Sentinel，可以使丢失写的窗口限制在某些时刻，然而这里也还有其他更不安全的方法来部署它.

    You need Sentinel support in your clients. Popular client libraries have Sentinel support, but not all.
    There is no HA setup which is safe if you don't test from time to time in development environments, or even better if you can, in production environments, if they work. You may have a misconfiguration that will become apparent only when it's too late (at 3am when your master stops working).
    Sentinel, Docker, or other forms of Network Address Translation or Port Mapping should be mixed with care: Docker performs port remapping, breaking Sentinel auto discovery of other Sentinel processes and the list of replicas for a master. Check the section about Sentinel and Docker later in this document for more information.



