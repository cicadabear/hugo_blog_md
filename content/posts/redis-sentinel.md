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
你至少需要三个Sentinel实例，来实现一个健壮的部署.

2. The three Sentinel instances should be placed into computers or virtual machines that are believed to fail in an independent way. So for example different physical servers or Virtual Machines executed on different availability zones.  
三个Sentinel实例应该被放在三台不同的物理机或者虚拟机，在三个不同的可用地区执行.  

3. Sentinel + Redis distributed system does not guarantee that acknowledged writes are retained during failures, since Redis uses asynchronous replication. However there are ways to deploy Sentinel that make the window to lose writes limited to certain moments, while there are other less secure ways to deploy it.  
Sentinel + Redis分布式系统不会保证接收的写入会被保留在出现错误的时候，既然Redis使用异步复制. 然而，这里有方法来部署Sentinel，可以使丢失写的窗口限制在某些时刻，然而这里也还有其他更不安全的方法来部署它.

4. You need Sentinel support in your clients. Popular client libraries have Sentinel support, but not all. 
你需要使用支持Sentinel的客户端

5. There is no HA setup which is safe if you don't test from time to time in development environments, or even better if you can, in production environments, if they work. You may have a misconfiguration that will become apparent only when it's too late (at 3am when your master stops working).  
这里没有高可用又安全的设置，如果你不在测试环境不时的测试，如果你能在正式环境测试更好. 你可能有一个错误配置当它出现的时候为时已晚（凌晨3点你的master停止工作）.

6. **Sentinel, Docker, or other forms of Network Address Translation or Port Mapping should be mixed with care:** Docker performs port remapping, breaking Sentinel auto discovery of other Sentinel processes and the list of replicas for a master. Check the section about Sentinel and Docker later in this document for more information.  
Sentinel, Docker, 或者其他形式的网络地址转换或者事端口映射，需要小心混合: docker执行端口映射，阻碍了Sentinel自动发现其他的Sentinel进程，和一个master的replica实例. 检查关于Sentinel和Docker的章节来获取更多信息。

### Configuring Sentinel  

The Redis source distribution contains a file called sentinel.conf that is a self-documented example configuration file you can use to configure Sentinel, however a typical minimal configuration file looks like the following:  
Redis的源发布包含一个sentinel.conf文件，这是一个带有文档的示例配置文件，你可以用来配置Sentinel，然而一个一般最小的配置文件是像下边这样：

```
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 60000
sentinel failover-timeout mymaster 180000
sentinel parallel-syncs mymaster 1

sentinel monitor resque 192.168.1.3 6380 4
sentinel down-after-milliseconds resque 10000
sentinel failover-timeout resque 180000
sentinel parallel-syncs resque 5
```

You only need to specify the masters to monitor, giving to each separated master (that may have any number of replicas) a different name. There is no need to specify replicas, which are auto-discovered. Sentinel will update the configuration automatically with additional information about replicas (in order to retain the information in case of restart). The configuration is also rewritten every time a replica is promoted to master during a failover and every time a new Sentinel is discovered.  

你只需要指定要监控的masters, 给与每个不同的master（每个master可能有多个replica实例）一个不同的名字。在这里不需要指定replica实例，replica实例是自动发现的。带着replica实例的附加信息，Sentinel将会自动更新配置（用来保存信息如果重启）。每次出错当一个replica实例被提升为master实例的时候或者一个新的Sentinel被发现的时候，这个配置也会被重写。

The example configuration above basically monitors two sets of Redis instances, each composed of a master and an undefined number of replicas. One set of instances is called ```mymaster```, and the other ```resque```.  
这个示例配置文件超过了基本的监控两组Redis实例，每一组包含了一个master实例和未定义数量的replica实例。其中一组叫作 ```mymaster```，另一组叫作```resque```。

The meaning of the arguments of ```sentinel monitor``` statements is the following:  

这个配置文件里边参数的含义如下：

    sentinel monitor <master-group-name> <ip> <port> <quorum>

For the sake of clarity, let's check line by line what the configuration options mean:  
为了更清楚，我们一行一会的检查，配置选项的含义：  

The first line is used to tell Redis to monitor a master called mymaster, that is at address 127.0.0.1 and port 6379, with a quorum of 2. Everything is pretty obvious but the **quorum** argument:  

第一行用来告诉Redis来监控一个master, 叫作mymaster, 它监听着127.0.0.1的6379端口，quorum参数是2，除了quorum之外，别的都是显而易见的：  

* The **quorum** is the number of Sentinels that need to agree about the fact the master is not reachable, in order to really mark the master as failing, and eventually start a failover procedure if possible.  
quorum参数是所需要的Sentinel的数量，来同意master实例不可用的事实，为了真的标记master实例失败，并且如果可能，最终启动一个容错进程。

* However **the quorum is only used to detect the failure**. In order to actually perform a failover, one of the Sentinels need to be elected leader for the failover and be authorized to proceed. This only happens with the vote of the **majority of the Sentinel processes**.  
然而quorum参数只是被用来探测错误的。为了实际上执行容错，Sentinel中的一个需要被选出来为容错和授权执行作领导。这个只会是多数Sentinel进程投票决定。  

So for example if you have 5 Sentinel processes, and the quorum for a given master set to the value of 2, this is what happens:  

所以，例如你有5个Sentinel进程，并且对于一个给定的master, quorum参数是2, 以下就是会发生的：  

* If two Sentinels agree at the same time about the master being unreachable, one of the two will try to start a failover.  
如果两个Sentinel在同一时间同意master实例不可访问，这两个Sentinel中的一个就会尝试启动容错进程。

* If there are at least a total of three Sentinels reachable, the failover will be authorized and will actually start.  
如果这里至少有总共3个Sentinel可访问，容错将会被授权启动。

In practical terms this means during failures **Sentinel never starts a failover if the majority of Sentinel processes are unable to talk** (aka no failover in the minority partition).
实际上这里的含义是，在出错的时候，如果多数的Sentinel不能访问，容错将不会启动，也就是在少数Sentinel部分，容错不会启动。

Other Sentinel options 

其他Sentinel选项

The other options are almost always in the form:  
其他的选项几乎都是以下形式:  

    sentinel <option_name> <master_name> <option_value>

And are used for the following purposes:
被用于以下目的：  

* ```down-after-milliseconds``` is the time in milliseconds an instance should not be reachable (either does not reply to our PINGs or it is replying with an error) for a Sentinel starting to think it is down.  
```down-after-milliseconds```是Sentinel判断一个实例是否不可用的最大无响应时间（PING无回应或者回应错误）

* ```parallel-syncs``` sets the number of replicas that can be reconfigured to use the new master after a failover at the same time. The lower the number, the more time it will take for the failover process to complete, however if the replicas are configured to serve old data, you may not want all the replicas to re-synchronize with the master at the same time. While the replication process is mostly non blocking for a replica, there is a moment when it stops to load the bulk data from the master. You may want to make sure only one replica at a time is not reachable by setting this option to the value of 1.  
```parallel-syncs```设置了replica实例可以同时在容错之后被重新配置使用新master的实例数量（同时被重新配置的relica实例数量）。这个数值越小，容错进程运行时间越长，然而如果这个relica实例被配置成提供老的数据，你可能就不想重新同时同步所有replica实例了。然而复制进程多数情况下是非阻塞的对于一个replica实例，但是这里有一个时刻，当它停止去加载master的大块数据。你可能想要确认同一个时刻只有一个replica实例不可访问，通过设置这个选项为1.

Additional options are described in the rest of this document and documented in the example ```sentinel.conf``` file shipped with the Redis distribution.  

附加的选项在下文被描述并且在```sentinel.conf```中也有文档，和Redis发行版在一起。

All the configuration parameters can be modified at runtime using the SENTINEL SET command. See the **Reconfiguring Sentinel at runtime** section for more information.  

所有的配置参数都可以在运行时被修改，使用```SENTINEL SET```命令。看**Reconfiguring Sentinel at runtime**这个章节，为获取更多相关信息。

Example Sentinel deployments

Sentinel部署示例  

Now that you know the basic information about Sentinel, you may wonder where you should place your Sentinel processes, how many Sentinel processes you need and so forth. This section shows a few example deployments.  

现在你已经知道了Sentinel的基本信息，你或许想知道，你应该把Sentinel进程放在哪里，你需要多少个Sentinel进程之类的问题。这个章节展示了几个示例部署。  

We use ASCII art in order to show you configuration examples in a graphical format, this is what the different symbols means:  

我们使用ASCII艺术用图形的形式来展示配置示例，这就是不同的符号代表着什么：  

这是一个会独立失败的物理机或者虚拟机，我们称它为box.  

```
+--------------------+
| This is a computer |
| or VM that fails   |
| independently. We  |
| call it a "box"    |
+--------------------+
```  

We write inside the boxes what they are running:  
我们在box的内部写上他们运行着什么：  

```
+-------------------+
| Redis master M1   |
| Redis Sentinel S1 |
+-------------------+
```

Different boxes are connected by lines, to show that they are able to talk:  
不同的box之间连着线，表示他们之间可以相互交流：  

```
+-------------+               +-------------+
| Sentinel S1 |---------------| Sentinel S2 |
+-------------+               +-------------+
```

Network partitions are shown as interrupted lines using slashes:  
网络分区展示为被斜线中断的线：  

```
+-------------+                +-------------+
| Sentinel S1 |------ // ------| Sentinel S2 |
+-------------+                +-------------+
```

Also note that:  
还要注意这些：  

* Masters are called M1, M2, M3, ..., Mn.  
Master实例叫作M1, M2, M3, ..., Mn.  

* replicas are called R1, R2, R3, ..., Rn (R stands for replica).  
Replica实例叫作R1, R2, R3, ..., Rn.  

* Sentinels are called S1, S2, S3, ..., Sn.  
Sentinels叫作S1, S2, S3, ..., Sn.  

* Clients are called C1, C2, C3, ..., Cn.  
Clients叫作C1, C2, C3, ..., Cn.  

* When an instance changes role because of Sentinel actions, we put it inside square brackets, so [M1] means an instance that is now a master because of Sentinel intervention.  
当一个实例因为Sentinel动作改变角色的时候，我们把它放在方括号中，所以[M1]是一个master实例，因为Sentinel的介入。  

Note that we will never show setups where just two Sentinels are used, since Sentinels always need to talk with the majority in order to start a failover.  

注意我们将不会展示两个Sentinel实例使用的设置，既然Sentinel总是需要多数交流来开启一次容错。  

## Example 1: just two Sentinels, DON'T DO THIS  

## 示例 1： 只有两个Sentinel实例，不要这样做  

```
+----+         +----+
| M1 |---------| R1 |
| S1 |         | S2 |
+----+         +----+

Configuration: quorum = 1  
```

* In this setup, if the master M1 fails, R1 will be promoted since the two Sentinels can reach agreement about the failure (obviously with quorum set to 1) and can also authorize a failover because the majority is two. So apparently it could superficially work, however check the next points to see why this setup is broken.  
在这个设置中，如果master M1出错了，R1 将会被提升既然两个Sentinel可以就出错达成共识（quorum设置为1，显而易见的）并且，还可以授权一次容错，因为多数是2。所以很显然表面上它可以工作，然而检查下一个点来看看为什么这个设置会是坏的。  

* If the box where M1 is running stops working, also S1 stops working. The Sentinel running in the other box S2 will not be able to authorize a failover, so the system will become not available.  
如果M1的box停止工作了，S1也会停止工作。运行在另一个box的Sentinel将不会授权一次容错，所以整个系统就不可用了。  

Note that a majority is needed in order to order different failovers, and later propagate the latest configuration to all the Sentinels. Also note that the ability to failover in a single side of the above setup, without any agreement, would be very dangerous:  

注意需要一个大多数来给不同的容错下命令，并且之后传播最新的配置到所以的Sentinel上。还要注意只用一个Sentinel来决定是否master不可用，而不需要任何的同意，是非常危险的。  

```
+----+           +------+
| M1 |----//-----| [M1] |
| S1 |           | S2   |
+----+           +------+
```

In the above configuration we created two masters (assuming S2 could failover without authorization) in a perfectly symmetrical way. Clients may write indefinitely to both sides, and there is no way to understand when the partition heals what configuration is the right one, in order to prevent a permanent split brain condition.  

在上边的配置中，我们创建了两个master(假设S2可以不用授权容错)，以一个对称的方式。客户端可能会不确定的写入到两边，这里无法理解当分区恢复的时候，哪个配置才是正确的一个，为了避免一个永久的大脑分裂状态。  

So please **deploy at least three Sentinels in three different boxes always.**
所以永远至少部署三个Sentinel实例在三个不同的box. 

### Example 2: basic setup with three boxes

This is a very simple setup, that has the advantage to be simple to tune for additional safety. It is based on three boxes, each box running both a Redis process and a Sentinel process.  

这是一个非常简单的设置，其优点是易于调整以提高安全性。它是基于三个box, 每一个box运行着Redis进程和Sentinel进程。  

```
       +----+
       | M1 |
       | S1 |
       +----+
          |
+----+    |    +----+
| R2 |----+----| R3 |
| S2 |         | S3 |
+----+         +----+

Configuration: quorum = 2
```

If the master M1 fails, S2 and S3 will agree about the failure and will be able to authorize a failover, making clients able to continue.  

如果M1出错了，S2和S3会同意这个错误，并且授权一次容错，使得客户端可以继续连接。  

In every Sentinel setup, as Redis uses asynchronous replication, there is always the risk of losing some writes because a given acknowledged write may not be able to reach the replica which is promoted to master. However in the above setup there is an higher risk due to clients being partitioned away with an old master, like in the following picture:  

在每一个Sentinel设置中，因为Redis使用了异步复制，这里永远会有丢失部分写入的风险，因为一个确认的写入，或许不能到达replica, 正好这个replica实例被提升为了master. 然而，在上边的配置中，有个跟高的风险，由于客户端和老的master一起被分入了跟其他Redis不同的分区，就像下图：  

```

         +----+
         | M1 |
         | S1 | <- C1 (writes will be lost)
         +----+
            |
            /
            /
+------+    |    +----+
| [M2] |----+----| R3 |
| S2   |         | S3 |
+------+         +----+
```

In this case a network partition isolated the old master M1, so the replica R2 is promoted to master. However clients, like C1, that are in the same partition as the old master, may continue to write data to the old master. This data will be lost forever since when the partition will heal, the master will be reconfigured as a replica of the new master, discarding its data set.  

在这个例子中，一个网络分区隔离了老的master M1, 所以R2被提升为master. 然而，客户端，比如C1, 跟老的master在一个分区，或许会向老的master继续写入。这里的数据将会永远丢失，当分区恢复的时候，这个maser将会被重新配置为新master的replica，丢弃自己的数据。  

This problem can be mitigated using the following Redis replication feature, that allows to stop accepting writes if a master detects that it is no longer able to transfer its writes to the specified number of replicas.  

这个问题可以通过使用以下Redis复制功能得以缓和，这种复制方法允许停止接收吸入，如果一个master探测到它不能够把写入传输到一定数量的replica上了。  

```
min-replicas-to-write 1
min-replicas-max-lag 10
``` 

With the above configuration (please see the self-commented redis.conf example in the Redis distribution for more information) a Redis instance, when acting as a master, will stop accepting writes if it can't write to at least 1 replica. Since replication is asynchronous not being able to write actually means that the replica is either disconnected, or is not sending us asynchronous acknowledges for more than the specified max-lag number of seconds.  

使用上述配置，一个Redis实例是master的时候，会停止接收写入，如果它不能写入至少一个replica. 既然复制是异步的，不能够写入的意思是，replica或者是断开连接了，或者是没能在指定的最大延迟时间内发回异步接收确认。  

Using this configuration, the old Redis master M1 in the above example, will become unavailable after 10 seconds. When the partition heals, the Sentinel configuration will converge to the new one, the client C1 will be able to fetch a valid configuration and will continue with the new master.  
使用这个配置，上述实例中的老的Redis master M1，将会在10秒钟后变得不可用。当分区恢复了，Sentinel配置将会使用新的，客户端C1将会获取一个有效的配置，并且和新的master继续工作。

However there is no free lunch. With this refinement, if the two replicas are down, the master will stop accepting writes. It's a trade off.  

然而这里没有免费的午餐，用这个配置，如果两个replica都宕机了，master将停止接收写入。这是一个权衡。

### Example 3: Sentinel in the client boxes

Sometimes we have only two Redis boxes available, one for the master and one for the replica. The configuration in the example 2 is not viable in that case, so we can resort to the following, where Sentinels are placed where clients are:  

有时我们只有两个Redis box可用，一个master一个replica. 这个示例2中的配置在这种情况下是不可行的，所以我们采用如下方法，Sentinel放在哪里，客户端放在哪里： 

```
            +----+         +----+
            | M1 |----+----| R1 |
            |    |    |    |    |
            +----+    |    +----+
                      |
         +------------+------------+
         |            |            |
         |            |            |
      +----+        +----+      +----+
      | C1 |        | C2 |      | C3 |
      | S1 |        | S2 |      | S3 |
      +----+        +----+      +----+

      Configuration: quorum = 2
```

In this setup, the point of view Sentinels is the same as the clients: if a master is reachable by the majority of the clients, it is fine. C1, C2, C3 here are generic clients, it does not mean that C1 identifies a single client connected to Redis. It is more likely something like an application server, a Rails app, or something like that.  

在这个设置中，Sentinel和客户端的视角一样：如果一个master对于多数客户端可用，它就是好的。C1, C2, C3这里都是通用的客户端，这并不意味着C1识别为一个单独连接到Redis的客户端. 它更像一个应用。

If the box where M1 and S1 are running fails, the failover will happen without issues, however it is easy to see that different network partitions will result in different behaviors. For example Sentinel will not be able to setup if the network between the clients and the Redis servers is disconnected, since the Redis master and replica will both be unavailable.  

如果一个box, 里边的M1和S1都出错了，容错将会启动，没有问题，然而很容易来看不同的网络分区将会导致不同的行为。比如，Sentinel将会不能设置，如果客户端和Redis服务器之间的网络断了，既然Redis Master和Replica都会不可用。  

Note that if C3 gets partitioned with M1 (hardly possible with the network described above, but more likely possible with different layouts, or because of failures at the software layer), we have a similar issue as described in Example 2, with the difference that here we have no way to break the symmetry, since there is just a replica and master, so the master can't stop accepting queries when it is disconnected from its replica, otherwise the master would never be available during replica failures.  

注意，如果C3和M1分在了同一个分区（上边说的网络情况出现的可能性很小，但是更可能的是不同的布局，或者是软件层的错误），我们有一个和示例2描述中类似的问题，带着这些不同，我们无法打破对称，既然这里只有一个replica和master, 所以master不能停止接收查询，当它从replica断开的时候，否则master将不可用，当replica出错的时候。  

So this is a valid setup but the setup in the Example 2 has advantages such as the HA system of Redis running in the same boxes as Redis itself which may be simpler to manage, and the ability to put a bound on the amount of time a master in the minority partition can receive writes.  

所以这是个有效的设置，但是示例2中的设置有优势，比如Redis高可用系统和Redis本事运行在同一个box可能更易于管理，并且限制在少数分区的master可以接收写入的时间。

### Example 4: Sentinel client side with less than three clients  
### 示例4：Sentinel客户端方面少于三个客户端

The setup described in the Example 3 cannot be used if there are less than three boxes in the client side (for example three web servers). In this case we need to resort to a mixed setup like the following:  

示例3中的设置，不能被使用，如果这里有少于三个box在客户端这边（例如三个web server）. 在这个情况下，我们需要采用混合设置的方法，如下：  

```

            +----+         +----+
            | M1 |----+----| R1 |
            | S1 |    |    | S2 |
            +----+    |    +----+
                      |
               +------+-----+
               |            |
               |            |
            +----+        +----+
            | C1 |        | C2 |
            | S3 |        | S4 |
            +----+        +----+

      Configuration: quorum = 3
```

This is similar to the setup in Example 3, but here we run four Sentinels in the four boxes we have available. If the master M1 becomes unavailable the other three Sentinels will perform the failover.  

这个跟示例3中的设置很相似，但是这里我们运行四个Sentinel在四个box中。如果master M1变得不可用，其他的三个Sentinel将会执行一次容错。  

In theory this setup works removing the box where C2 and S4 are running, and setting the quorum to 2. However it is unlikely that we want HA in the Redis side without having high availability in our application layer.  

理论上来说，这个设置在去掉最后一个box并且设置quorum为2的情况下可以工作. 然而，我们想要高可用系统在Redis一边，而没用高可用在应用层，这样不太可能。

## Sentinel, Docker, NAT, and possible issues

Docker uses a technique called port mapping: programs running inside Docker containers may be exposed with a different port compared to the one the program believes to be using. This is useful in order to run multiple containers using the same ports, at the same time, in the same server.  

Docker使用的技术叫作端口映射: 运行在docker容器里的程序，可能会暴露一个跟程序自己认为不同的端口。这个功能很实用当使用相同的端口运行多个容器的时候，同时，同一台服务器。  

Docker is not the only software system where this happens, there are other Network Address Translation setups where ports may be remapped, and sometimes not ports but also IP addresses.  

Docker不是仅有会出现这种情况的软件，这里还要其他的NAT设置会让端口重新映射，并且有时候不仅仅是端口，IP也会。

Remapping ports and addresses creates issues with Sentinel in two ways:  
重新映射端口和地址以如下两种方式给Sentinel造成了问题：  

* Sentinel auto-discovery of other Sentinels no longer works, since it is based on hello messages where each Sentinel announce at which port and IP address they are listening for connection. However Sentinels have no way to understand that an address or port is remapped, so it is announcing an information that is not correct for other Sentinels to connect.  
Sentinel自动发现其他的Sentinel的功能不再工作，因为它是基于Sentinel播报的hello message，这条信息中包含了自己的的端口和IP. 然而，Sentinel无法理解IP或者端口被重新映射了，所以它们正在播报不正确的IP和端口供其他Sentinel来连接。  

* Replicas are listed in the INFO output of a Redis master in a similar way: the address is detected by the master checking the remote peer of the TCP connection, while the port is advertised by the replica itself during the handshake, however the port may be wrong for the same reason as exposed in point 1.  
Replicas被列出来在一个Redis master的INFO输出上，以一个类似的方法：IP地址被master探测通过检查连接的远程端，然而端口被replica自己通知在握手的时候，然而端口可能是错误的因为和上边第一点相同的原因。  
















