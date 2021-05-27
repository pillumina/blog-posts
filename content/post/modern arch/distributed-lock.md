---
title: "微服务：分布式锁方案"
date: 2021-05-26T14:22:18+08:00
hero: /images/posts/k8s-docker.jpg
keywords: ["microservice", "distributed lock"]
tags: ["microservice"]
categories: ["microservice"]
menu:
  sidebar:
    name: 微服务中的分布式锁方案
    identifier: distributed-lock
    parent: system-architecture
    weight: 10
draft: false
---

## 为什么需要分布式锁？

在分布式系统中，常常需要协调他们的动作。如果不同的系统或是同一个系统的不同主机之间共享了一个或一组资源，那么访问这些资源的时候，往往需要互斥来防止彼此干扰进而保证一致性，这个时候，便需要使用到分布式锁。

## 常见的实现方案

### 基于数据库实现

#### 数据表方案

最容易想到的一种实现方案就是基于数据库的。可以在数据库中创建一张锁表，然后通过操作该表中的数据来实现。

可以这样创建一张表：

```sql
create table TDistributedLock (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `lock_key` varchar(64) NOT NULL DEFAULT '' COMMENT '锁的键值',
  `lock_timeout` datetime NOT NULL DEFAULT NOW() COMMENT '锁的超时时间',
  `create_time` datetime NOT NULL DEFAULT NOW() COMMENT '记录创建时间',
  `modify_time` datetime NOT NULL DEFAULT NOW() COMMENT '记录修改时间',
  PRIMARY KEY(`id`),
  UNIQUE KEY `uidx_lock_key` (`lock_key`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='分布式锁表';
```



这样在需要加锁时，只需要往这张表里插入一条数据就可以了：

```sql
INSERT INTO TDistributedLock(lock_key, lock_timeout) values('lock_xxxx', '2020-07-19 18:20:00');
```

当对共享资源的操作完毕后，可以释放锁：

```sql
DELETE FROM TDistributedLock where lock_key='lock_xxxx';
```



该方案简单方便，主要利用数据库表的唯一索引约束特性，保证多个微服务模块同时申请分布式锁时，只有一个能够获得锁。

下面讨论下一些相关问题：

1. 获得锁的微服务模块意外crash，来不及释放锁怎么办？因为在插入锁记录时，同时设置了一个`lock_timeout`属性，可以另外跑一个`lock_cleaner`，将超时的锁记录删除。当然为了安全，`lock_timeout`最好设置一个合理的值，以确保在这之后正常的共享资源操作一定是完成了的。
2. 获得锁失败的微服务模块如何继续尝试获得锁？搞一个while循环，反复尝试，如能成功获得锁就跳出循环，否则sleep一会儿重新进入循环体继续尝试。
3. 数据库单点不安全怎么办？数据库领域有主从、一主多从、多主多从等复制方案，可保证一个数据库实例挂掉时，其它实例可以接管过来继续提供服务。不过需要注意的是有一些复制方案它是异步的，并不能保证写入一个数据库实例的数据立马可以在另外一个数据库实例中查询到， 这就容易造成锁丢失，导致授予了两把锁的问题。
4. 获得锁的微服务模块想重入地获得锁怎么办？在数据库表中加个字段，记录当前获得锁的机器的主机信息和线程信息，那么下次再获取锁的时候先查询数据库，如果当前机器的主机信息和线程信息在数据库可以查到的话，直接把锁分配给它就可以了。同样为了确保安全，必须在本微服务模块没有主动释放锁，同时离锁的超时时间还很久远的情况下，才可以安全地直接把锁再次分配给它，同时更新锁的超时时间。

该方案的缺陷：

1. 该方案利用数据库表自身的唯一键约束，性能相比后面提到的redis方案会差一点。
2. 该方案中其它尝试获得锁的客户端会反复尝试插入数据，消耗数据库资源。
3. 该方案中需要为数据库找一种比较可靠的高可用方案，同时还得确保数据库实例之间复制方案是满足一致性要求的。

#### 数据库表的行排它锁方案

除了动态地往数据库表里插入数据外，还可以预先将锁信息写入数据库表，然后利用数据库的行排它锁来进行加锁与释放锁操作。

例如加锁时可以执行以后命令：

```sql
SET autocommit = 0;
START TRANSACTION;
SELECT * FROM TDistributedLock WHERE lock_key='lock_xxxx' FOR UPDATE;
```

当对共享资源的操作完毕后，可以释放锁：

```sql
COMMIT;
```

该方案也很简单，利用数据库表的行排它锁特性，保证多个微服务模块同时申请分布式锁时，只有一个能够获得锁。

下面讨论下一些相关问题：

1. 获得锁的微服务模块意外crash，来不及释放锁怎么办？没有`COMMIT`，数据库会自动将锁释放掉。
2. 获得锁失败的微服务模块如何继续尝试获得锁？`for update`语句会在执行成功后立即返回，在执行失败时一直处于阻塞状态，直到成功，因此不用写while循环反复尝试获得锁了。
3. 数据库单点问题跟上面那个方案一致，不再赘述。
4. 在这个方案下，好像很难做到锁可重入了。

该方案的缺陷：

1. 该方案利用数据库表的行排它锁特性，性能上比上面那个方案好一点，但相比后面提到的redis方案还是差一点。
2. 该方案中需要为该表的数据库事务配置较高的事务超时时间，而且一个排他锁长时间不提交，会占用数据库连接。
3. 该方案中同样需要为数据库找一种比较可靠的高可用方案，同时还得确保数据库实例之间复制方案是满足一致性要求的。

### 基于缓存redis实现

使用redis做分布式锁的思路大概是这样的：在redis中设置一个值表示加了锁，然后释放锁的时候就把这个key删除。

具体代码如下：

```
获取锁
-- NX是指如果key不存在就成功，key存在返回false，PX可以指定过期时间
redis.call("set", key, ARGV[1], "NX", "PX", ARGV[2])
释放锁
-- 释放锁涉及到两条指令，这两条指令不是原子性的
-- 需要用到redis的lua脚本支持特性，redis执行lua脚本是原子性的
if redis.call("get", key) == ARGV[1] then
	return redis.call("del", key)
else
	return 0
end
```

上述的lua代码比较简单，不具体解释了。这里要注意给锁键的value值要保证唯一，这个是为了避免释放错了锁。场景如下：假设A获取了锁，过期时间30s，此时35s之后，锁已经自动释放了，A去释放锁，但是此时可能B获取了锁。加上释放锁前的value值判断，A客户端就不能删除B的锁了。

如果是javascript的项目，可以使用[redislock](https://github.com/danielstjules/redislock)库，它封装了上述加锁、释放锁等操作逻辑，用起来很方便。

才区区几行代码就优雅地搞定了分布式锁，而且redis的写入、删除键值的性能超高，看样子很完美，但事实上并非如此。

#### 键的超时时间

在redis中设置键时可以通过`PX`指定过期时间，这个时间不宜设置得太大，否则万一获得锁的进程crash了，要等很久此键才能过期自动删除。这个时间也不宜设得过小，否则对共享资源的操作还没完成，锁就释放了，其它服务就又能获得锁了。这里可以采用watchdog之类的方案，获得锁时设置一个较小的超时时间，然后在持有锁的过程中定期对锁的租期进行延长。

#### redis的高可用架构

为了解决redis的单点问题，一般在生产环境会为redis实施高可用架构方案。可问题是redis主从复制方案均是异步的，在主从切换过程中有可能造成锁丢失。Redis的作者`Antirez`为此提出了一个[RedLock的算法方案](https://redis.io/topics/distlock)，这个算法的大概逻辑如下：

假设存在多个Redis实例，这些节点是完全独立的，不需要使用复制或者任何协调数据的系统，多个redis实例中获取锁的过程就变成了如下步骤：

1. 以毫秒为单位获取当前的服务器时间
2. 尝试使用相同的key和随机值来获取锁，对每一个机器获取锁时都应该有一个超时时间，比如锁的过期时间为10s，那么获取单个节点锁的超时时间就应该为5到50毫秒左右，他这样做的目的是为了保证客户端与故障的机器连接不耗费多余的时间！超时间时间内未获取数据就放弃该节点，从而去下一个节点获取，直至将所有节点全部获取一遍！
3. 获取完成后，获取当前时间减去步骤一获取的时间，当且仅当客户端半数以上获取成功且获取锁的时间小于锁额超时时间，则证明该锁生效！
4. 获取锁之后，锁的超时时间等于设置的有效时间-获取锁花费的时间
5. 如果获取锁的机器不满足半数以上，或者锁的超时时间计算完毕后为负数等异常操作，则系统会尝试解锁所有实例，即使有些实例没有获取锁成功，依旧会被尝试解锁！
6. 释放锁，只需在所有实例中释放锁，无论客户端是否认为它能够成功锁定给定的实例。

这个方案看似很好，但仔细审视后还是发现一些问题的，分布式架构师`Martin`就提出了自己的[意见](http://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)。这些意见总结下来如下：

> 分布式锁的用途无非两种：
>
> - 要么为了提升效率，用锁来保证一个任务没有必要被执行两次。比如（很昂贵的计算） 保证正确，使用锁来保证任务按照正常的步骤执行，防止两个节点同时操作一份数据，造成文件冲突，数据丢失。这种情况下对锁是有一定宽容度的，就算发生了两个节点同时工作，对系统的影响也仅仅是多付出了一些计算的成本，没什么额外的影响。这个时候 使用单点的 Redis 就能很好的解决问题，没有必要使用RedLock，维护那么多的Redis实例，提升系统的维护成本。
> - 要么为了安全性，此时必须从理论上确保拿到的锁是安全的，两个进程不可能同时持有某把锁，因为很可能涉及到一些金钱交易，如果锁定失败，并且两个节点同时处理同一数据，则结果将导致文件损坏，数据丢失，永久性不一致，或者金钱方面的损失！而RedLock方案从理论上说并不能保证这一点。

`RedLock`方案从理论上说并不能保证锁的安全性，主要有以下几点原因：

1. 锁的自动过期机制很可能导致锁的意外丢失。因此采用watchdog机制确保在持有锁之后持续地给锁续期是有必要的。

2. ```
   RedLock
   ```

   方案对于系统时钟有强依赖。假设有A、B、C、D、E 5个redis节点：

   1. 客户端1获取节点A，B，C的锁定。由于网络问题，无法访问D和E。
   2. 节点C上的时钟向前跳，导致锁过期。
   3. 客户端2获取节点C，D，E的锁定。由于网络问题，无法访问A和B。
   4. 现在，客户1和2都认为他们持有该锁。

3. ```
   RedLock
   ```

   方案同样无法避免redis实例意外重启导致的问题。假设有A、B、C、D、E 5个redis节点：

   1. 客户端1获取节点A，B，C的锁定。由于网络问题，无法访问D和E。
   2. 节点C上的redis实例意外重启，重启后原来写入的键值丢失。
   3. 客户端2获取节点C，D，E的锁定。由于网络问题，无法访问A和B。
   4. 现在，客户1和2都认为他们持有该锁。

虽说`RedLock`从理论上说确实无法100%保证锁的安全性，但以上列举的场景极为严苛，事实上在现实中很难碰到。而由于该方案获取锁的效率确实很高，事实上还是有不少业务场景就是使用的该方案。

如果是javascript的项目，配合着使用[timers](https://github.com/component/timers)库，即可实施watchdog方案，在持有锁的过程中定期对锁的租期进行延长。如果要使用`RedLock`方案，可以使用[node-redlock](https://github.com/mike-marcacci/node-redlock)库，它封装了上述`RedLock`方案的复杂逻辑，用起来也很方便。

如果对锁的安全性要求极高，真的不允许任何锁安全性问题，还可以试试下面的zookeeper方案。

### 基于zookeeper实现

`zookeeper`是一种提供配置管理、分布式协同以及命名的中心化服务。很明显`zookeeper`本身就是为了分布式协同这个目标而生的，它采用一种被称为ZAB(Zookeeper Atomic Broadcast)的一致性协议来保证数据的一致性。基于zk的一些特性，我们很容易得出使用zk实现分布式锁的落地方案：

1. 使用zk的临时节点和有序节点，每个线程获取锁就是在zk创建一个临时有序的节点，比如在/lock/目录下。
2. 创建节点成功后，获取/lock目录下的所有临时节点，再判断当前线程创建的节点是否是所有的节点的序号最小的节点
3. 如果当前线程创建的节点是所有节点序号最小的节点，则认为获取锁成功。
4. 如果当前线程创建的节点不是所有节点序号最小的节点，则对节点序号的前一个节点添加一个事件监听。 比如当前线程获取到的节点序号为`/lock/003`,然后所有的节点列表为`[/lock/001,/lock/002,/lock/003]`,则对`/lock/002`这个节点添加一个事件监听器。

如果锁释放了，会唤醒下一个序号的节点，然后重新执行第3步，判断是否自己的节点序号是最小。

比如`/lock/001`释放了，`/lock/002`监听到时间，此时节点集合为`[/lock/002,/lock/003]`,则`/lock/002`为最小序号节点，获取到锁。

从数据一致性的角度来说，zk的方案无疑是最可靠的，而且等候锁的客户端也不用不停地轮循锁是否可用，当锁的状态发生变化时可以自动得到通知。

但zookeeper实现也存在较大的缺陷：

- 性能问题

zookeeper作为分布式协调系统，不适合作为频繁的读写存储系统。而且通过增加zookeeper服务器来提高集群的读写能力也是有上限的，因为zookeeper集群规模越大，zookeeper数据需要同步到更多的服务器。同时zookeeper分布式锁每一次都要申请锁和释放锁，都要动态创建删除临时节点，所以zookeeper不能维护大量的分布式锁，也不能维护大量的客户端心跳长连接。在分布式定理中，zookeeper追求的是CP，也就是zookeeper保证集群向外提供统一的视图，但是zookeeper牺牲了可用性，在极端情况下，zookeeper可能会丢弃一些请求。并且zookeeper集群在进行leader选举的时候整个集群也是不可用的，集群选举时间长达30 ~ 120s。

- 惊群效应

在获取锁的时候有一个细节，客户端在获取锁失败的情况下，会监听`/lock`节点。这会存在性能问题，如果在`/lock`节点下排队等待的有1000个进程，那么锁持有者释放锁(删除`/lock`节点下的临时节点)时，zookeeper会通知监听`/lock`的1000个进程。然后这1000个进程会读取zookeeper下`/lock`节点的全部临时节点，然后判断自己是否为最小的临时节点。但是这1000个进程中只有一个最小序号的进程会持有分布式锁，也就是说999个进程做的都是无用功。这些无用功会对zookeeper造成较大压力的读负载。为了解决惊群效应，需要对zookeeper分布式锁监听逻辑进行优化，实际上，排队进程真正感兴趣的是比自己临时节点序号小的节点，我们只需要监听序号比自己小的节点。

另外还需要注意的是，使用zookeeper方案也不是说就高枕无忧了。假设某一个客户端通过上述方案获得了锁，但由于网络问题，该客户端与zookeeper集群间失去了联系，zookeeper的心跳无效后，该客户端会收到了一个zookeeper的SessionTimeout的事件。为了保证分布式锁的有效性，这个时候客户端就需要在下一个等侯者获得锁之前，中断对共享资源的访问，然后继续尝试获取锁。

如果是javascript的项目，可以使用[zk-lock](https://github.com/metamx/zk-lock)库，它封装了上述方案的复杂逻辑，用起来也很方便。

## 总结

总的来说，目前分布式锁领域暂时没有出现十分完美、无懈可击的方案。个人觉得，综合对比下，还是推荐采用缓存redis方案。如果项目较小，影响面不大，采用单实例redis就差不多了。如果对redis的高可用有要求，可以采用`RedLock`方案，配合`watchdog`机制定期延长key的续约租期，不过要通过合理的部署、运维等方式规避系统时钟、网络分区问题。如果真的对分布锁有着极高的一致性要求，同时对锁的性能不太在意的话，也可以采用zookeeper方案。



## 参考

1. http://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html
2. https://redis.io/topics/distlock