## PART1 系统容灾

🐮:<b>Redis是基于内存的存储，如果服务重启，数据不就丢失了吗？</b>

🐶:可以通过<font color=blue>持久化机制，备份数据</font>。有两种方式，一种是开启RDB，RDB是Redis的二进制快照文件，优点是文件紧凑，占用空间小，恢复速度比较快。同时，由于是子进程Fork的模式，对Redis本身读写性能的影响很小。
<center>
  
![](https://files.mdnice.com/user/13621/144b4125-d2c1-416c-833e-2b44039031ea.png)</center>

🐶:另一种方式是AOF，AOF中记录了Redis的操作命令，可以重放请求恢复现场，AOF的文件会比RDB大很多。

🐶:出于性能考虑，如果开启了AOF，会将命令先记录在AOF缓冲，之后再刷入磁盘。数据刷入磁盘的时机根据参数决定，有三种模式：<b><font color=blue>1.关闭时刷入；2.每秒定期刷入；3.执行命令后立刻触发。</font></b>

🐶:AOF的优点是故障情况下，丢失的数据会比RDB更少。如果是执行命令后立马刷入，AOF会拖累执行速度，所以一般都是配置为每秒定期刷入，这样对性能影响不会很大。
<center>
  
![](https://files.mdnice.com/user/13621/42a77ecb-dd1b-47e0-9b4f-4e4bb97f2f35.png)</center>

🐮:<b>这样看起来，AOF文件会越来越大，最后磁盘都装不下？</b>

🐶:不会的，Redis可以在AOF文件体积变得过大时，自动地在后台Fork一个子进程，专门对AOF进行重写。说白了，就是针对相同Key的操作，进行合并，比如同一个Key的set操作，那就是后面覆盖前面。

🐶:在重写过程中，Redis不但将新的操作记录在原有的AOF缓冲区，而且还会记录在AOF重写缓冲区。一旦新AOF文件创建完毕，Redis 就会将重写缓冲区内容，追加到新的AOF文件，再用新AOF文件替换原来的AOF文件。
<center>
  
![](https://files.mdnice.com/user/13621/13a4a5b5-f5e8-415f-a43c-a6cdc6b4cb0e.png)</center>

🐮:<b>Redis机器挂掉怎么办？
</b>

🐶:可以用主从模式部署，即有一个或多个备用机器，备用机会作为Slave同步Master的数据，在Redis出现问题的时候，把Slave升级为Master。
<center>

![](https://files.mdnice.com/user/13621/507b60e5-89d5-44a2-92bf-dc44a4d4a1a1.png)</center>

🐮:<b>主从可以自动切换吗？
</b>

🐶:本身是不能，但可以写脚本实现，只是需要考虑的问题比较多。Redis已经有了现成的解决方案：<b><font color=blue>哨兵模式</font></b>。哨兵来监测Redis服务是否正常，异常情况下，由哨兵代理切换。为避免哨兵成为单点，哨兵也需要多机部署。
<center>

![](https://files.mdnice.com/user/13621/66bec3be-edc9-4224-9384-113ff40860ab.png)</center>

🐮:<b>如果Master挂掉，会选择哪个Slave呢？</b>

🐶:当哨兵集群选举出哨兵Leader后，由哨兵Leader从Redis从节点中选择一个Redis节点作为主节点：

🐶:1.过滤故障的节点；

🐶:2.选择优先级slave-priority最大的从节点作为主节点，如不存在，则继续；

🐶:3.选择复制偏移量最大的从节点作为主节点，如果都一样，则继续。这里解释下，数据偏移量记录写了多少数据，主服务器会把偏移量同步给从服务器，当主从的偏移量一致，则数据是完全同步；

🐶:4.选择runid最小的从节点作为主节点。Redis每次启动的时候生成随机的runid作为Redis的标识。
<center>

![](https://files.mdnice.com/user/13621/7aab46b1-9ce3-4388-b26a-3de54c03a480.png)</center>

🐮:<b>前面你提到了哨兵Leader，那它是怎么来的呢？</b>

🐶:每一个哨兵节点都可以成为Leader，当一个哨兵节点确认Redis集群的主节点主观下线后，会请求其他哨兵节点要求将自己选举为Leader。被请求的哨兵节点如果没有同意过其他哨兵节点的选举请求，则同意该请求，也就是选举票数+1，否则不同意。

🐶:如果一个哨兵节点获得的选举票数<b><font color=blue>超过节点数的一半，且大于quorum配置的值</font></b>，则该哨兵节点选举为Leader；否则重新进行选举。
<center>

![](https://files.mdnice.com/user/13621/0ae14e65-d4db-4d70-b84e-2a91bb298c16.png)</center>

## PART2 性能优化

🐮:<b>Redis性能怎么样？</b>

🐶:只能说在十万级。使用之前，要跑BenchMark，实际情况下会受带宽、负载、单个数据大小、是否开启多线程等因素影响，<font color=blue>脱开具体场景谈性能，就没有意义</font>。
<center>

![](https://files.mdnice.com/user/13621/781a4036-3feb-4f27-9d7e-5d5fc0003bad.png)</center>

🐮:<b>Redis性能这么高，那它是协程模型，还是多线程模型？</b>

🐶:Redis是单线程Reactor模型，通过<font color=blue>高效的IO复用</font>以及<font color=blue>内存处理</font>实现高性能。如果是6.0之前我会毫不犹豫说是单线程，6.0之后，我还是会说单线程，但会补充一句，IO解包通过多线程进行了优化，而处理逻辑，还是单线程。

🐶:另外，如果考虑到RDB的Fork，一些定时任务的处理，那么Redis也可以说多进程，这没有问题。但是Redis对数据的处理，至始至终，都是单线程。
<center>

![](https://files.mdnice.com/user/13621/57d5f26c-a04a-4114-b572-263b50ebaf5e.png)</center>

🐮:<b>可以详细说下6.0版本发布的多线程功能吗？</b>

🐶:多线程功能，主要用于提高解包的效率。和传统的Multi Reactor多线程模型不同，Redis的多线程只负责处理网络IO的解包和协议转换，一方面是因为Redis的单线程处理足够快，另一方面也是为了兼容性做考虑。
<center>

![](https://files.mdnice.com/user/13621/4ef622eb-6b22-4f60-9822-ec80855dfabe.png)</center>

🐮:<b>如果数据太大，Redis存不下了怎么办？</b>

🐶:<b><font color=blue>使用集群模式</font></b>。也就是将数据分片，不同的Key根据Hash路由到不同的节点。集群索引是通过一致性Hash算法来完成，这种算法可以解决服务器数量发生改变时，所有的服务器缓存在同一时间失效的问题。

🐶:同时，基于Gossip协议，集群状态变化时，如新节点加入、节点宕机、Slave提升为新Master，这些变化都能传播到整个集群的所有节点并达成一致。
<center>

![](https://files.mdnice.com/user/13621/15949b4a-cea4-4ee2-baa9-443d6da85465.png)</center>

🐮:<b>一致性Hash能详细讲一下吗？</b>

🐶:好的，传统的Hash分片，可以将某个Key，落到某个节点。但有一个问题，当节点扩容或者缩容，路由会被完全打乱。如果是缓存场景，很容易造成缓存雪崩问题。
<center>

![](https://files.mdnice.com/user/13621/59db6ac1-4cc6-4248-8c41-1a8cfb9c3b24.png)</center>

🐶:为了优化这种情况，一致性Hash就应运而生了。一致性Hash是说将数据和服务器，以相同的Hash函数，映射到同一个Hash环上，针对一个对象，在哈希环上顺时针查找距其最近的机器，这个机器就负责处理该对象的相关请求。

🐶:这种情况下，增加节点，只会分流后面一个节点的数据。减少节点，那么请求会由后一个节点继承。也就是说，节点变化操作，最多只会影响后面一个节点的数据。
<center>

![](https://files.mdnice.com/user/13621/5f7251d7-a409-4d0f-a18b-a4a87beef966.png)</center>

🐮:<b>好了，看你对Redis掌握还不错，下面我们来聊聊场景相关的应用吧。</b>

### 牛牛复盘
Redis是后台开发的核心技术，是工作中的必备技能，本次我们聚焦于Redis的高级主题，容灾和性能，这两项都是拉开能力差距的项，如果想拿到比较好的offer，一定要重点准备。祝小伙伴们都拿到心仪的Offer，我们下期再见。

