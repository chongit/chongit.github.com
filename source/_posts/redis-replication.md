title: Redis Replication
layout: post
date: 2013-12-13 17:11:54
category : tech
tagline: "Redis Doc"
description: Redis的主从实现
tags : [redis,replication,translate]
---
## Replication ##

[原文地址][1]

Replication功能可以通过一个简单的配置实现master--slave的复制，它允许slave节点从master节点中抽取复制数据。

以下是replication非常重要的特点：

 - redis使用异步的复制。从2.8开始，slaves会周期性（每秒一次）的发送acknowledge。
 - 一个master可以有多个slave。
 - slave可以接受其他slave的连接。除了连接多个slave到同一个相同master，slave也可以连接其他slave以此构成一个图的结构。
 - Replication在master端是非阻塞，这意味着master在其slaves提交第一个同步时也可以继续执行查询。
 - Replication在slave端是非阻塞的，slave可以在提交第一个同步的同时使用旧的数据提供查询，确保你在redis.conf中配置了这个功能。另外你可以配置当master连接不可以用（down掉）是slave返回error给客户端。然而，在旧数据集必须删除和新数据集必须加载时会有block客户端的链接的情况。
 - Replication可以同时用于可扩展性，以便有多个slave进行只读查询，例如，sort操作可以交给到slave来分担，或只是为数据容灾。
 - 可以使用复制避免在master端的保存过程：只需要配置master的`redis.conf`避免存储（注释掉所有的`save`选项），然后不时的连接一个配置了`save`的slave。


### Replication怎么工作 ###
如果你安装了一个slave，不论他是第一次连接还是重连，连接后它会马上发送一个`SYNC`的命令。
接着master启动后台存储，并收集所有接收到可以修改数据集的的新指令。

当后端的存储完成，master把数据文件传输给slave，slave会先保存在硬盘上然后加载进内存中。再然后master将会发送所有积累的命令，和所有从客户端收到的新的用来修改数据集的命令。这些通过和redis协议相同格式的命令流的方式完成。

你可以通过`telnet`尝试。在该服务器正在做一些工作时连接redis端口并发送`SYNC`指令。你会看到一个批量传输，然后所有master收到的指令会在telnet回话中重现。

当master因为一些原因down掉时slave可以自动重连。如果master同步收到slave的同步请求，它只执行一个后台存储来提供给所有slave。

当一个master和一个slave断开后重连，将会提交一个全同步。然而redis2.8以后，可以支持块重同步*(partial resynchronization)*。

### 块同步 ###
从redis2.8开始，master和slave通常可以在断开重连后继续复制进程而不需要请求全同步。

这个工作在master端为复制流使用了内存backlog，master和所有的slave保持同步的`replication offset`和`master run id`，所以当连接中断，slave重连并请求master继续复制，假设`master run id`没有变，并且`replication backlog`的offset可用的，如果条件满足，master只发送部分它丢失的复制流，并且复制继续。然而，在老版本的redis中会执行全同步。

新的*partial resynchronization*功能内部使用`PSYNC`命令，老的实现使用`SYNC`，然而，一个redis2.8的slave可以检测服务器是否支持`PSYNC`，否则使用`SYNC`。

### 配置 ###
配置replication很简单，只需要在slave的配置文件中添加如下行：

    slaveof 192.168.1.1 6379

当然你需要替换掉`192.168.1.1 6379`而使用你的主机`IP地址（或域名）`和`端口`。或者，你可以执行 `SLAVEOF`，master主机就会开始一个同步。
也有一些参数用来调整master内存中的replication backlog来执行块同步。参见随redis发布的redis.conf示例了解更多信息。

从redis2.6开始slave默认支持只读模式。这个行为被redis.conf里的“slave-read-only”选项控制。而且可以在运行中使用CONFIG SET启用或关闭。

只读模式的slave会拒绝所有的写命令，所以即使在错误的情况下slave也不可能被写入。这并不意味着该同能被设想为把slave示例暴露在互联网或更普遍的存在不可信客户端的网络中。因为类似DEBUG和CONFIG的管理命令依旧可用。但是只读实例的安全性可以通过在redis.conf中使用重命名指令禁用命令而提高。

你可能想知道为什么它可以恢复默认的，并有slave的实例，可以是目标写入操作。其原因是，虽然这个写入将被丢弃，如果从站和主会重新同步，或者如果从重新启动，常常有临时数据是不重要的，可以存储到slave。例如客户端可能需要提供关于slave到master的可达性信息来协调故障切换策略。


### 设置slave到master的认证 ###
如果你的master通过requirepass有密码认证，也可以很简单的配置salve在所有sync操作中使用密码认证。
在正在运行的实例上，使用redis-cli和类型：

    config set masterauth <password>
    
永久的设置，在你的配置文件中添加

    masterauth <password>


### Allow writes only with N attached replicas ###
从redis2.8起，为了提高数据安全性，可以配置一个redis master接受写查询在至少n个slave已连接到master时。
然而，因为redis使用异步复制所以不可能确保接受的写入被确保实际写入，所以这成为丢失数据的一个窗口。    

这是功能原理：

 - redis slave每秒钟ping master，回应replication流被处理的数量。
 - redis的master会记住它最后一次收到ping的时间。
 - 用户可以配置一个之后不超过几秒钟延迟的slave的最大数目。

如果至少有n个slave，延迟了小于m秒，就会接受写操作。

你可以在它认为作为一个宽松的版的“C”在CAP定理，其中一致性不能保证对于给定的写，但至少在时间窗口的数据丢失被限制为给定秒数。

如果条件不具备，master会使用一个错误回复和写将不被接受代替回复。

关于此功能的两个参数：
    
    min-slaves-to-write <number of slaves>
    min-slaves-max-lag <number of seconds>


欲了解更多信息，请随Redis的源代码分发的例子redis.conf文件。


  [1]: http://redis.io/topics/replication