---
title:      " Redis高可用总结：Redis主从复制、哨兵集群、脑裂... "
date:       2021-02-10 19:37
banner_img: "https://gitee.com/like-ycy/images/raw/master/blog/header.jpg"
tags: [ Redis ]

---

在实际的项目中，服务高可用非常重要，如，当Redis作为缓存服务使用时， 缓解数据库的压力，提高数据的访问速度，提高网站的性能 ，但如果使用Redis 是单机模式运行 ，只要一个服务器宕机就不可以提供服务，这样会可能造成服务效率低下，甚至出现其相对应的服务应用不可用。



因此为了实现高可用，Redis 提供了哪些高可用方案？

- Redis主从复制
- Redis持久化
- 哨兵集群
- ...

![](https://gitee.com/like-ycy/images/raw/master/blog/2021-02-10/1.png)

Redis基于一个Master主节点多Slave从节点的模式和Redis持久化机制，将一份数据保持在多个实例中实现增加副本冗余量，又使用哨兵机制实现主备切换， 在master故障时，自动检测，将某个slave切换为master，最终实现Redis高可用 。



------



### **Redis主从复制**

Redis主从复制，主从库模式一个Master主节点多Slave从节点的模式，将一份数据保存在多Slave个实例中，增加副本冗余量，当某些出现宕机后，Redis服务还可以使用。

![](https://gitee.com/like-ycy/images/raw/master/blog/2021-02-10/2.png)

但是这会存在数据不一致问题，那redis的副本集是如何数据一致性？



Redis为了保证数据副本的一致，主从库之间采用读写分离的方式：

- **读操作：主库、从库都可以执行处理；**
- **写操作：先在主库执行，再由主库将写操作同步给从库。**

![](https://gitee.com/like-ycy/images/raw/master/blog/2021-02-10/3.png)

使用读写分离方式的好处，可以避免当主从库都可以处理写操作时，主从库处理写操作加锁等一系列巨额的开销。



采用读写分离方式，写操作只会在主库中进行后同步到从库中，那主从库是如何同步数据的呢？



主从库是同步数据方式有两种：

- **全量同步：**通常是主从服务器刚刚连接的时候，会先进行全量同步
- **增量同步 ：**一般在全同步结束后，进行增量同步，比如主从库间网络断，再进行数据同步。

####  

#### **全量同步**

主从库间第一次全量同步，具体分成三个阶段：

- 当一个从库启动时，从库给主库发送 psync 命令进行数据同步（psync 命令包含：主库的 runID 和复制进度 offset 两个参数），
- 当主库接收到psync 命令后将会保存RDB 文件并发送给从库，发送期间会使用缓存区（replication buffer）记录后续的所有写操作 ，从库收到数据后，会先清空当前数据库，然后加载从主库获取的RDB 文件，
- 当主库完成 RDB 文件发送后，也会把将保存发送RDB文件期间写操作的replication buffer发给从库，从库再重新执行这些操作。这样一来，主从库就实现同步了。

![](https://gitee.com/like-ycy/images/raw/master/blog/2021-02-10/4.png)

另外，为了分担主库生成 RDB 文件和传输 RDB 文件压力，提高效率，可以使用 “主 - 从 - 从”模式将主库生成 RDB 和传输 RDB 的压力，以级联的方式分散到从库上。

![](https://gitee.com/like-ycy/images/raw/master/blog/2021-02-10/5.png)



#### **增量同步**

增量同步，基于环形缓冲区repl_backlog_buffer缓存区实现。



在环形缓冲区，主库会记录自己写到的位置 master_repl_offset ，从库则会记录自己已经读到的位置slave_repl_offset, 主库并通过master_repl_offset 和 slave_repl_offset的差值的数据同步到从库。

![](https://gitee.com/like-ycy/images/raw/master/blog/2021-02-10/6.png)

主从库间网络断了， 主从库会采用增量复制的方式继续同步，主库会把断连期间收到的写操作命令，写入 replication buffer，同时也会把这些操作命令也写入 repl_backlog_buffer 这个缓冲区，然后主库并通过master_repl_offset 和 slave_repl_offset的差值数据同步到从库。

![](https://gitee.com/like-ycy/images/raw/master/blog/2021-02-10/7.png)

因为repl_backlog_buffer 是一个环形缓冲区，当在缓冲区写满后，主库会继续写入，此时，会出现什么情况呢？



覆盖掉之前写入的操作。如果从库的读取速度比较慢，就有可能导致从库还未读取的操作被主库新写的操作覆盖了，这会导致主从库间的数据不一致。因此需要关注 repl_backlog_size参数，调整合适的缓冲空间大小，避免数据覆盖，主从数据不一致。



主从复制，除了会出现数据不一致外，甚至可能出现主库宕机的情况，Redis会有主从自主切换机制，那如何实现的呢？




-----



### **Redis哨兵机制**

当主库挂了，redis写操作和数据同步无法进行，为了避免这样情况，可以在主库挂了后重新在从库中选举出一个新主库，并通知到客户端，redis提供了 哨兵机制，哨兵为运行在特殊模式下的 Redis 进程。



#### **Redis会有主从自主切换机制，那如何实现的呢？**



哨兵机制是实现主从库自动切换的关键机制，其主要分为三个阶段:

- **监控：哨兵进程会周期性地给所有的主从库发送 PING 命令，检测它们是否仍然在线运行。**
- **选主（选择主库）：主库挂了以后，哨兵基于一定规则评分选选举出一个从库实例新的主库 。**
- **通知 ：哨兵会将新主库的信息发送给其他从库，让它们和新主库建立连接，并进行数据复制。同时，哨兵会把新主库的信息广播通知给客户端，让它们把请求操作发到新主库上。**

![](https://gitee.com/like-ycy/images/raw/master/blog/2021-02-10/8.png)

**其中，在监控中如何判断主库是否处于下线状态？**



哨兵对主库的下线判断分为：

- 主观下线：**哨兵进程会使用 PING 命令检测它自己和主、从库的网络连接情况，用来判断实例的状态**， 如果单哨兵发现主库或从库对 PING 命令的响应超时了，那么，哨兵就会先把它标记为“主观下线”
- 客观下线：在哨兵集群中，基于少数服从多数，多数实例都判定主库已“主观下线”，则认为主库“客观下线”。



**为什么会有这两种"主观下线"和“客观下线”的下线状态呢？**

由于单机哨兵很容易产生误判，误判后主从切换会产生一系列的额外开销，为了减少误判，避免这些不必要的开销，采用哨兵集群，引入多个哨兵实例一起来判断，就可以避免单个哨兵因为自身网络状况不好，而误判主库下线的情况，



基于少数服从多数原则， 当有 N 个哨兵实例时，最好要有 N/2 + 1 个实例判断主库为“主观下线”，才能最终判定主库为“客观下线” （可以自定义设置阙值）。

![](https://gitee.com/like-ycy/images/raw/master/blog/2021-02-10/9.png)

**那么哨兵之间是如何互相通信的呢？**

哨兵集群中哨兵实例之间可以相互发现，**基于 Redis 提供的发布 / 订阅机制（pub/sub 机制）**,



哨兵可以在主库中发布/订阅消息，在主库上有一个名为“\__sentinel__:hello”的频道，不同哨兵就是通过它来相互发现，实现互相通信的，而且只有订阅了同一个频道的应用，才能通过发布的消息进行信息交换。

![](https://gitee.com/like-ycy/images/raw/master/blog/2021-02-10/10.png)

哨兵 1连接相关信息（IP端口）发布到“\__sentinel__:hello”频道上，哨兵 2 和 3 订阅了该频道。



哨兵 2 和 3 就可以从这个频道直接获取哨兵 1连接信息，以这样的方式哨兵集群就形成了，实现各个哨兵互相通信。



哨兵集群中各个实现通信后，就可以判定主库是否已客观下线。



**在已判定主库已下线后，又如何选举出新的主库？**



新主库选举按照**一定条件**筛选出的符合条件的从库，并按照**一定规则**对其进行打分，最高分者为新主库。



通常**一定条件**包括：

- 从库的当前在线状态，
- 判断它之前的网络连接状态，通过down-after-milliseconds * num(断开连接次数)，当断开连接次数超过阈值，不适合为新主库。



**一定规则包括**

- 从库优先级 ， 通过slave-priority 配置项，给不同的从库设置不同优先级，优先级最高的从库得分高
- 从库复制进度，和旧主库同步程度最接近的从库得分高，通过repl_backlog_buffer缓冲区记录主库 master_repl_offset 和从库slave_repl_offset 相差最小高分
- 从库 ID 号 ， ID 号小的从库得分高。



**全都都基于在只有在一定规则中的某一轮评出最高分从库就选举结束，哨兵发起主从切换。**



#### **leader哨兵**

**选举完新的主库后，不能每个哨兵都发起主从切换，需要选举成leader哨兵，那如何选举leader哨兵执行主从切换？**



选举leader哨兵，也是基于少数服从多数原则"投票仲裁"选举出来，

- 当任何一个从库判定主库“主观下线”后，发送命令 s-master-down-by-addr命令发送想要成为Leader的信号，
- 其他哨兵根据与主机连接情况作出相对的响应，赞成票Y，反对票N，而且如果有多个哨兵发起请求，每个哨兵的赞成票只能投给其中一个，其他只能为反对票。



想要成为Leader 的哨兵，要满足两个条件：

- 第一，获得半数以上的赞成票；
- 第二，获得的票数同时还需要大于等于哨兵配置文件中的quorum值。



**选举完leader哨兵并新主库切换完毕之后，那么leader哨兵怎么通知客户端？**



还是基于哨兵自身的 pub/sub 功能，实现了客户端和哨兵之间的事件通知，客户端订阅哨兵自身消息频道 ，而且哨兵提供的消息订阅频道有很多，不同频道包含了：



| 事件         | 相关频道                                                     |
| ------------ | ------------------------------------------------------------ |
| 主库下线事件 | +sdown（实例进入“主观下线”状态） -sdown（实例退出“主观下线”状态） +odown（实例进入“客观下线”状态） -odown（实例退出“客观下线”状态） |
| 新主库切换   | + switch-master（主库地址发生变化）                          |



其中，当客户端从哨兵订阅消息主从库切换，当主库切换后，端户端就会接收到新主库的连接信息：

```
switch-master <master name> <oldip> <oldport> <newip> <newport>
```

在这样的方式哨兵就可以通知客户端切换了新库。



基于上述的机制和原理Redis实现了高可用，但也会带了一些潜在的风险，比如数据缺失。



------



### **数据问题**

Redis实现高可用，但实现期间可能产出一些风险：

- **主备切换的过程， 异步复制导致的数据丢失**
- **脑裂导致的数据丢失**
- **主备切换的过程，异步复制导致数据不一致**

####  

#### **数据丢失-主从异步复制**

因为master 将数据复制给slave是异步实现的，在复制过程中，这可能存在master有部分数据还没复制到slave，master就宕机了，此时这些部分数据就丢失了。



**总结：主库的数据还没有同步到从库，结果主库发生了故障，未同步的数据就丢失了。**



#### **数据丢失-脑裂**

何为脑裂？当一个集群中的 master 恰好网络故障，导致与 sentinal 通信不上了，sentinal会认为master下线，且sentinal选举出一个slave 作为新的 master，此时就存在两个 master了。



此时，可能存在client还没来得及切换到新的master，还继续写向旧master的数据，当master再次恢复的时候，会被作为一个slave挂到新的master 上去，自己的数据将会清空，重新从新的master 复制数据，这样就会导致数据缺失。



**总结：主库的数据还没有同步到从库，结果主库发生了故障，等从库升级为主库后，未同步的数据就丢失了。**



#### **数据丢失解决方案**

数据丢失可以通过合理地配置参数 min-slaves-to-write 和 min-slaves-max-lag 解决，比如

- min-slaves-to-write 1
- min-slaves-max-lag 10



如上两个配置：要求至少有 1 个 slave，数据复制和同步的延迟不能超过 10 秒，如果超过 1 个 slave，数据复制和同步的延迟都超过了 10 秒钟，那么这个时候，master 就不会再接收任何请求了。



#### **数据不一致**

在主从异步复制过程，当从库因为网络延迟或执行复杂度高命令阻塞导致滞后执行同步命令，这样就会导致数据不一致



**解决方案：可以开发一个外部程序来监控主从库间的复制进度（master_repl_offset 和 slave_repl_offset ），通过监控 master_repl_offset 与slave_repl_offset差值得知复制进度，当复制进度不符合预期设置的Client不再从该从库读取数据。**

![](https://gitee.com/like-ycy/images/raw/master/blog/2021-02-10/11.png)



### **总结**

Redis使用主从复制、持久化、哨兵机制等实现高可用，需要理解其实现过程，也要明白其带了风险以及解决方案，才能在实际项目更好优化，提升系统的可靠性、稳定性。
