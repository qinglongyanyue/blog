---
layout: default
title:  "FAST论文分析——HopsFS"
date:   2017-05-14
categories: paper
---

一直在关注文件系统领域，对于将文件系统元数据存在各类DB也是有点兴趣，关注到的论文包括CMU早期发表的TableFS，IndexFS，已经近期FAST15和16年的BetrFS，主要是基于key-value DB来加速元数据存储。

这篇论文则更进一步，考虑用NewSQL来存放元数据，不知道FS与NewSQL能擦出什么样的火花呢？来看看这篇文章吧。

## 1. 背景

任何论文最重要的就是他的出发点，要解决啥问题，看看本文作者看到的痛点是啥？

随着NewSQL DB的快速发展，分布式DB的性能，扩展性，事务性能，新的内存介质，使得我们又得考虑将分布式FS的元数据架在NewSQL之上啦。 

NewSQL多数都是以Google Spanner为蓝本，比如国内做的不错的TiDB，国外的cockroachDB都获得了极大的关注，以及学术界Yale大学的CalvinDB。其实Yale的Alexander Thomson教授在FAST2015就有篇CalvinFS就是基于CalvinDB设计的分布式FS，其中更关注跨地域的分布式FS。

这篇论文更多的关注的集群内部的元数据管理，并且以HDFS的元数据管理为例，阐明了作者的方案的价值。。

作者来自瑞典名校皇家工学院，合作者有oracle的专家，看样子oracle有资助这个事情，虽然并不是传统的存储名校，但是主题关系比较大，还是值得分析下。

作者认为当前有几个痛点：

>
1. 当前的分布式FS基本都是为大文件设计的，能通过将数据切割个不同节点并发提升性能，但是对于文件系统元数据的处理就很挫了。
2. 仍有不少分布式FS将元数据放在单点，比如HDFS；或者将元数据分布式打散放在多个DISK中，比如GPFS，GFS等；或者直接简单将元数据sharding称不同的名字空间，每个名字空间的元数据独立管理。
3. 现有各种方案都不够完美，如果把元数据放到一个牛逼的分布式数据库搞定多好，支持事务，横向扩展，in-memory处理，SQL语义。但是FS的元数据是树形结构，如何能高效的映射到数据中呢？本文的核心就是搞定这个。

总体的设计思路通过改造HDFS实现，名为HopsFS，整个方案的核心思路如下：

>
1. 将HDFS的元数据存储和服务管理动作分离；使用NewSQL DB存放所有的元数据，NDB，后台是一个基于MySQL的分布式数据库；HopsFS在上层通过多个无状态的节点并发的去访问数据库，通过并发提升性能。
2. HopsFS将文件系统的元数据操作包装为分布式事务，提交给NDB执行。并通过多种方式提升性能：
  - Batch多个操作，即每次事务尽量并发的发多个req下去
  - 开启write ahead cache，关于同一个事务的元数据先在内存聚合下再发下去
  - 通过NewSQL与FS的联合设计，比如FS将同一个子目录存放到NewSQL的同一个Partition或者sharding中，以及将事务尽量放在一个节点完成，以及提供高效的scan接口，方便ls类操作。
  - 提供一个inode hint cache，避免每次全路径的寻址开销。
3. 对于少量的元数据操作，确实可以走数据的事务，但是低于mv和delete大目录这种同时需要巨多元数据的操作，直接使用数据的分布式事务显然是不现实的。
  - 对于这类操作，论文引入了一种新的方法：使用应用级的分布式锁机制去隔离这种子树级的操作。在加锁之后，就可以使用数据库的事务去一小部分小部分的操作了。
4. HopsFS直接hack了HDFS，且在生产环境使用，且对于元数据写密集型负载，性能提升37倍。

## 2. HopsFS整体架构
![]({{ site.baseurl }}/assets/hopsfs-1.png)

整体架构非常简单：
- 原生的HDFS元数据部署方式:包括一个active的namenode，一个standby的namenode，至少3个journal node用来做日志存储（quorum算法），至少3个zk node用来做集群协同。搞了这么多事情，实际上上对外提供服务的只有active的namenode。。
- HopsFS的架构明显更加简单，无状态AA的多个namenode + 分布式数据库，性能好，逻辑简单。通过DB + nodeid选择一个当主（为啥不简单的用zk来选个主？），解决HDFS的心跳信息同步问题。
- 每个HopsFS的namenode都有一个Data access layer（类似JDBC），封装所有针对分布式数据库的操作。
- 客户端可以使用Round-Robin、random或者pin主的方案选择namenode；datanode汇报block的信息到leader namenode，leader再把block信息同步到所有其他namenode（这个信息量不大，可以简单这么玩）。

## 3. HopsFS分布式元数据
![]({{ site.baseurl }}/assets/hopsfs-2.png)
![]({{ site.baseurl }}/assets/hopsfs-3.png)

元数据设计的总体思路比较清晰：
- 一切从业务负载特征出发：从作者收集到的负载特征来看，或者业务公布的特征来看，list，file read和stat占据了95%的IO，如上面表格中的绿色填充。
- 看看底层分布式数据的擅长啥，不擅长啥：跨shard的全表扫描，index scan都是DB消耗非常大的操作，所以要尽量少的去跨shard干活。而在一个shard内部，index scan，primary key的batch操作，primary key的单key操作都是非常高效的，所以要多用这块的能力。后面的思路全是围绕这条路来的。

### 3.1 文件系统到DB的关系模型
![]({{ site.baseurl }}/assets/hopsfs-4.png)

如上图所示，一切信息都映射到DB的表中：

实体类型：
- inode实体：
- block实体：file的数据块
- 副本实体：block的位置实体

实体之间的关系：
- 目录inode直接对应inode表的一行，file inode除了自己的信息之后，还对应多个实体，包括块列表，每个块的位置以及checksum等，这些额外的实体需要额外的表来处理。
- 目录和文件都是inode表中的一个实体，这个实体中还包括父目录的inode信息。
- inode表中不是存放全路径，而是父目录inode + file name
- 每个file都会有1个或多个block，block的信息会在多个表中展示。（key应该是inode + block id？）
  - 1）正常的block table；
  - 2）有节点失效，某些block的副本失效，此时block在URB表（低于正常副本数的表中）；
  - 3）URB表中的副本需要加入到PRB（pending replication table）表中进行恢复；
  - 4）当副本损坏时，数据块在CR表中；
  - 5)正在修复中的损坏block在RUC表中；
  - 6）有时候，副本数量超过设置量，加入到ER表；
  - 7)删除中的ER表的实体会放在Inv表中。
- 很重要的一点:inode实体中还包括一个外键，这个外键代表inode所在的partition，确保这个这个inode的所有操作都在一个DB的节点完成。

### 3.2 元数据分区方案

先讨论没有热点的情况：
- HopsFS直接使用父目录的inode id去分区元数据；同一个父目录下的所有inode元数据放在一个shard
- file inode相关的block信息，副本位置，checksum通过fileid来partition
- 理想情况下（无热点），负载都在一个节点内部解决，基本没有跨节点的元数据操作，效率高

#### 3.2.1 热点问题

如果有热点，怎么处理呢？
- root肯定是最大的瓶颈，HopsFS直接在所有节点cache住root。且不支持修改root的任何元数据
- 第一层目录也可能会是访问热点，默认可以配置第一层随机打散，后面的目录尽量聚合到一起
- 当然也可以视负载特征把目录打散，但是会牺牲mv和ls的性能。

## 4. HopsFS事务操作
HopsFS的事务操作包括2类：
1. inode类：针对一个实体（file，dir，block）操作的动作，比如mknod，mkdir，block stat改变等
2. 子树类：针对未知个inode操作的动作（可能达到上百万个），比如recursive del，mv，chmod，chown对一个目录。

首先看看inode类操作如何映射到DB的操作，NDB（本文中的分布式mySql）最强的语义仅仅是read commit语义（[关于DB语义的细节可以参考链接](http://www.cnblogs.com/ego/articles/1456317.html)）,但是FS的很多操作是需要串行语义的，如何解决呢：
- 使用行锁解决inode操作的并发冲突，但是一个事务中申请多个锁可能引起死锁，那怎么解决呢？
  1. 将所有的加锁动作顺序化，从根开始使用左序深度优先算法
  2. HDFS还有一些对inode读操作紧跟着写这个inode，这个数据库中会造成死锁（先拿读锁，升级到写锁），HopsFS直接将这些操作用提前用写锁。

### 4.1 Inode hint cache

层层路径解析和check权限一直都是文件系统中最常见的元数据操作，如何加速这些操作呢？

论文使用hint机制加速路径解析：
- 每个name node都将将父目录inode + filename缓存起来（仅仅cache key），解析过程中可以并发的发起DB的查询操作。

#### 4.1.1 cache一致性
- 在文件系统操作之前，通过inode hint cache + DB的并发读取获取实际的inode信息，当cache失效时
- 主key的读操作会失败，此时需要到DB中一层层解析，解析完成刷新内存
- cache一般不会经常失效，因为mv以及inode更新在Hadoop场景并不多

### 4.2 inode操作
![]({{ site.baseurl }}/assets/hopsfs-5.png)

HopsFS从他的实际场景发现，大负载情况下，悲观并发设计的性能要高于乐观并发设计（即默认冲突比较多），所有的inode操作都三个独立的阶段，加锁，执行，update：
- 加锁阶段：
  1. 找到file对应的inode hint cache
  2. 找到含有本操作相关所有信息，或者绝大部分信息的节点；事务就在这个节点执行
  3. 并发去解析路径（直接用DB的Read commit语义， 不加锁）
  4. 如果cache不命中，则层层去解析目录
  5. 锁定并读取最后一个inode信息
  6. 以及操作的类型，读取需要的数据
  7. HopsFS使用层级锁的概念，对于一个数的根，或者子树的根，意味着对整个树或者子树加锁；对inode加锁意味着整个inode的所有相关信息加锁
- per-transactin cache：
  1. 事务过程中的所有读取到的信息都放在内存cache起来，避免多次访问DB
  2. 数据的一致性通过行锁解决，除非事务结束，否则其他事务禁止修改cache的内容
- 执行和update：
  1. 每个inode的事务都在内存完成，执行完成之后，进入update阶段，以batch的方式批量提交到DB中

## 5. 处理大操作
针对目录的recursive的操作，可能会涉及百万级的inode，目前的DB还没有支持给百万条目加锁的，所以无法在一个事务中处理这么多的请求。这些操作包括move，delet，chown，chmod，set quota等。

### 5.1 子树操作协议
由于DB无法支持那么大的事务，论文提出一种子树操作协议：
- 应用给子树加一个应用级分布式大锁，防止事务冲突
  1. 在子树操作结束之前，禁止任何新操作去访问这个子树
  2. 在子树操作启动的之间，树是静止的（即树上没有修改操作在同时进行）
  3. 出现错误时，不会出现孤儿inode或者不一致

![]({{ site.baseurl }}/assets/hopsfs-6.png)

子树操作协议包括以下的步骤：
1. 获取整个子树的root，加锁并持久化到DB，意味着这个子树全部加了写锁；加锁之前需要确认整个子树没有任何冲突的操作。有一个表存放所有加了锁的子树，方便查询锁冲突。
2. 为了让子树保持静止，子树操作需要在所有的子树操作结束之后再锁定整个子树;HopsFS通过一个线程池来并发的去DB中扫描inode状态，并读取inode到内存中，方便后续的操作。如果发现并发冲突，随机往后等一段时间在发起。
3. 将大目录操作切割为多个并发的事务去执行。（并发的事务如果有失败的怎么办？）加锁时从上往下，删除时从叶子往根，防止出现孤儿。

### 5.2 子树操作失败的处理

如果在子树操作的过程中，节点down掉，怎么处理？HopsFS使用一种lazy cleanup的方案
- 每个namenode通过主namenode获取到active的namenode列表
- 如果发现有一个subtree的锁属于一个dead namenode，那么这个锁直接回收掉。
- 如果在删除操作过程中，如果节点失效，那么更换其他active的节点继续执行操作；如果是mv，chown之类的动作，主要对根目录操作（DB保证事务），此时只会存在忘记释放key的问题；通过第一步的方式搞定。

## 6. 小结

整体来看，论文的核心是将HDFS的元数据全部卸载到分布式数据库，并抽象出3种实体，很多table来解决元数据的各种问题。对于我们后续开发设计有较强的实用价值。



