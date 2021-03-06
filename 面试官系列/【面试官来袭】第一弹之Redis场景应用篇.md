## PART1 场景应用

🐮:<b>Redis经常用作缓存，那数据一致性怎么保证？</b>

🐶:从设计思路来说，有Cache Aside和Read/Write Through两种模式，前者是把缓存责任交给应用层，后者是将缓存的责任，放置到服务提供方。
<center>

![](https://files.mdnice.com/user/13621/2927faa3-8273-4422-b7a2-002fe34f154f.png)</center>

🐮:<b>你说到两种模式，那么哪种模式更好呢？</b>

🐶:两种模式各有优缺点，<font color=blue>从透明性考虑</font>，服务方比较合适；如果<font color=blue>从性能极致来说</font>，业务方会更有优势，毕竟可以减去服务RPC的损耗。

🐮:<b>嗯，如果数据发生变化，你会怎样去更新缓存？</b>

🐶:更新方式的话，大概有四种：

🐶:1.数据存到数据库中，成功后，再让缓存失效，等到读缓存不命中的时候，再加载进去；

🐶:2.通过消息队列更新缓存；

🐶:3.先更新缓存，再更新服务，这种情况相当于把Cache也做Buffer用；

🐶:4.起一个同步服务，作为MySQL一个从节点，通过解析binlog同步重要缓存。
<center>

![](https://files.mdnice.com/user/13621/9bece930-be82-4d34-afff-196a42334614.png)</center>

🐮:<b>那我们来谈谈Redis缓存雪崩。</b>

🐶:<b><font color=blue>缓存雪崩</font></b>表示在某一时间段，缓存集中失效，导致请求全部走数据库，有可能搞垮数据库，使整个服务瘫痪。雪崩原因一般是由于缓存过期时间设置得相同造成的。
<center>

![](https://files.mdnice.com/user/13621/52fb5893-e66f-4e97-a852-f7c04c9ff992.png)</center>

🐶:针对这种情况，可以借鉴ETCD中Raft选举的优化，让过期时间随机化，避免同一批请求，在同一时间过期。另一方面，还可以业务层面容灾，为热点数据使用双缓存。

🐮:<b>那Redis缓存穿透又是什么？</b>

🐶:缓存穿透指请求数据库里面根本没有的数据，<font color=blue>高频请求不存在的Key</font>，有可能是正常的业务逻辑，但更可能的，是黑客的攻击。
<center>

![](https://files.mdnice.com/user/13621/a3fc5ce1-b998-44ee-ac0a-94a518d7cb62.png)</center>

🐶:可以用布隆过滤器来应对，布隆过滤器是一种比较巧妙的概率型数据结构，特点是高效地插入和查询，可以用来告诉我们 “<b><font color=blue>某样东西一定不存在或者可能存在</font></b>”。
<center>

![](https://files.mdnice.com/user/13621/d13c2430-94d4-4fb7-bf0e-86bb6cd247da.png)</center>

🐮:<b>那你说一下布隆过滤器的实现吧。</b>

🐶:布隆过滤器底层是一个64位的整型，将字符串用多个Hash函数映射不同的二进制位置，将整型中对应位置设置为1。

🐶:在查询的时候，如果一个字符串所有Hash函数映射的值都存在，那么数据可能存在。为什么说可能呢，就是因为其他字符可能占据该值，提前点亮。

🐶:可以看到，布隆过滤器优缺点都很明显，<font color=blue>优点是空间、时间消耗都很小，缺点是结果不是完全准确</font>。
<center>

![](https://files.mdnice.com/user/13621/b60ac88b-29fc-442f-9248-5c1668664f11.png)</center>

🐮:<b>还有个概念叫缓存击穿，你能讲讲看吗？</b>

🐶:嗯...缓存击穿，是指某一热键，被超高的并发访问，在失效的一瞬间，还没来得及重新产生，就有海量数据，直达数据库，可谓是兵临城下。

🐶:这种情况和缓存雪崩的不同之处，在于雪崩是大量缓存赶巧儿一起过期，击穿只是单个超热键失效。
<center>

![](https://files.mdnice.com/user/13621/be81c132-7c65-4c7b-82ad-f65829460eb7.png)</center>

🐶:这种超高频Key，我们要提高待遇，可以让它不过期，再单独实现数据同步逻辑，来维护数据的一致性。当然，无论如何，对后端肯定是需要限频的，不然如果Redis数据丢失，数据库还是会被打崩。<font color=blue>限频方式可以是分布式锁或分布式令牌桶</font>。

🐮:<b>那Redis可以做消息队列吗？</b>

🐶:嗯...可以是可以，但我觉得用它来做消息队列不合适。Redis本身没有支持AMQP规范，消息队列该有的能力缺胳膊少腿，消息可靠性不强。

🐶:因为总有人拿Redis做消息队列。Redis的作者都看不下去了，赶紧出了个Disque来专事专做，虽然没大红大紫，但至少明确告诉了我们，Redis，别拿来做消息队列！

🐮:<b>那你能谈谈Redis在秒杀场景的应用吗？</b>

🐶:Redis主要是起到<b><font color=blue>选拔流量</font></b>的作用，记录商品总数，还有就是已下单数，等达到总数之后拦截所有请求。可以多放些请求进来，然后塞入消息队列。

🐶:蚂蚁金服的云Redis提到消息队列可以用Redis来实现，但我觉得更好的方式是用Kafka这种标准消息队列组件。
<center>

![](https://files.mdnice.com/user/13621/a3b52cee-659d-4e3f-8e56-1dbf1dce2f37.png)</center>

🐮:<b>你能继续说说Redis在分布式锁中的应用吗？</b>

🐶:锁是计算机领域一个非常常见的概念，分布式锁也依赖存储组件，针对请求量的不同，可以选择Etcd、MySQL、Redis等。前两者可靠性更强，Redis性能更高。
<center>

![](https://files.mdnice.com/user/13621/48a90d94-7b98-4232-8ed4-43e281850129.png)</center>

🐮:<b>那我们再聊聊Redis在限流场景的应用吧。</b>

🐶:<font color=blue>在微服务架构下，限频器也需要分布式化</font>。无论是哪种算法，都可以结合Redis来实现。这里我比较熟悉的是基于Redis的分布式令牌桶。

🐶:很显然，Redis负责管理令牌，微服务需要进行函数操作，就向Redis申请令牌，如果Redis当前还有令牌，就发放给它。拿到令牌，才能进行下一步操作。

🐶:另一方面，令牌不光要消耗，还需要补充，出于性能考虑，可以<font color=blue>使用懒生成的方式</font>：使用令牌时，顺便生成令牌。这样子还有个好处：令牌的获取，和令牌的生成，都可以在一个Lua脚本中，保证了原子性。
<center>

![](https://files.mdnice.com/user/13621/ce320ffd-9bc6-4f48-af36-a6fd6cba6563.png)</center>

🐮:<b>可以了，你就是我们想要的仔。</b>

## PART2 面试点评
Redis是后台开发中非常重要的领域，更是面试环节的高频考点，本次的重点是Redis实战，考察的就是实际解决问题的经验和能力。如果是不具备实际项目经验的，那么就需要不怕麻烦地去做实验，这样才能培养这方面的能力。好了，祝小伙伴们都拿到心仪的Offer！


下期预告：牛牛面试官将携带<b><font color=blue>MySQL面试高频考点</font></b>卷土重来！



