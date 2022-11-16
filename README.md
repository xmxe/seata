1. 分布式事务
```
CAP理论告诉我们，一个分布式系统不可能同时满足一致(C:Consistency)，可用性(A: Availability)和分区容错性(P:Partition tolerance)这三个基本需求，最多只能同时满足其中的2个。

BASE：全称：Basically Available(基本可用)，Soft state（软状态）,和Eventually consistent（最终一致性）。

Base理论是对CAP中一致性和可用性权衡的结果，其来源于对大型互联网分布式实践的总结，是基于CAP定理逐步演化而来的。其核心思想是：既是无法做到强一致性（Strong consistency），但每个应用都可以根据自身的业务特点，采用适当的方式来使系统达到最终一致性(Eventual consistency)
```

2. 2PC
```
阶段一(准备阶段)：协调者向所有的参与者询问，是否准备好了执行事务，并开始等待各参与者的响应。执行事务各参与者节点执行事务操作，并将Undo和Redo信息记入事务日志中，各参与者向协调者反馈事务询问的响应，如果参与者成功执行了事务操作，那么就反馈给协调者Yes响应，表示事务可以执行；如果参与者没有成功执行事务，就返回No给协调者，表示事务不可以执行。

阶段二：在阶段二中，会根据阶段一的投票结果执行2种操作：执行事务提交，中断事务。

执行事务提交步骤如下：
发送提交请求：协调者向所有参与者发出commit请求。参与者收到commit请求后，会正式执行事务提交操作，并在完成提交之后释放整个事务执行期间占用的事务资源。参与者在完成事务提交之后，向协调者发送Ack信息。协调者接收到所有参与者反馈的Ack信息后，完成事务。

中断事务步骤如下：
发送回滚请求：协调者向所有参与者发出Rollback请求。参与者接收到Rollback请求后，会利用其在阶段一种记录的Undo信息来执行事务回滚操作，并在完成回滚之后释放在整个事务执行期间占用的资源。参与者在完成事务回滚之后，想协调者发送Ack信息。协调者接收到所有参与者反馈的Ack信息后，完成事务中断。

二阶段提交缺点：
1、同步阻塞问题。执行过程中，所有参与节点都是事务阻塞型的。当参与者占有公共资源时，其他第三方节点访问公共资源不得不处于阻塞状态。
2、单点故障。由于协调者的重要性，一旦协调者发生故障。参与者会一直阻塞下去。尤其在第二阶段，协调者发生故障，那么所有的参与者还都处于锁定事务资源的状态中，而无法继续完成事务操作。（如果是协调者挂掉，可以重新选举一个协调者，但是无法解决因为协调者宕机导致的参与者处于阻塞状态的问题）
3、数据不一致。在二阶段提交的阶段二中，当协调者向参与者发送commit请求之后，发生了局部网络异常或者在发送commit请求过程中协调者发生了故障，这回导致只有一部分参与者接受到了commit请求。而在这部分参与者接到commit请求之后就会执行commit操作。但是其他部分未接到commit请求的机器则无法执行事务提交。于是整个分布式系统便出现了数据不一致性的现象。
4、二阶段无法解决的问题：协调者再发出commit消息之后宕机，而唯一接收到这条消息的参与者同时也宕机了。那么即使协调者通过选举协议产生了新的协调者，这条事务的状态也是不确定的，没人知道事务是否被已经提交。
由于二阶段提交存在着诸如同步阻塞、单点问题、脑裂等缺陷，所以，研究者们在二阶段提交的基础上做了改进，提出了三阶段提交。
```

3. 3PC
```
与两阶段提交不同的是，三阶段提交有两个改动点。
引入超时机制。同时在协调者和参与者中都引入超时机制。
在第一阶段和第二阶段中插入一个准备阶段。保证了在最后提交阶段之前各参与节点的状态是一致的.
如果段时间内没有收到协调者的commit请求，那么就会自动进行commit，解决了2PC单点故障的问题。
```
4. TCC
```
2PC要求参与者实现了XA协议，通常用来解决多个数据库之间的事务问题，比较局限。在多个系统服务利用api接口相互调用的时候，就不遵守XA协议了，这时候2PC就不适用了。现代企业多采用分布式的微服务，因此更多的是要解决多个微服务之间的分布式事务问题。
TCC就是一种解决多个微服务之间的分布式事务问题的方案。TCC是Try、Confirm、Cancel三个词的缩写，其本质是一个应用层面上的2PC，同样分为两个阶段：
准备阶段：协调者调用所有的每个微服务提供的try接口，将整个全局事务涉及到的资源锁定住，若锁定成功try接口向协调者返回yes。
提交阶段：若所有的服务的try接口在阶段一都返回yes，则进入提交阶段，协调者调用所有服务的confirm接口，各个服务进行事务提交。如果有任何一个服务的try接口在阶段一返回no或者超时，则协调者调用所有服务的cancel接口

这里有个关键问题，既然TCC是一种服务层面上的2PC。它是如何解决2PC无法应对宕机问题的缺陷的呢？
答案是不断重试。
```

---

- [seata官方网站](http://seata.io/zh-cn/)
- [一致性协议算法-2PC、3PC、Paxos、Raft、ZAB、NWR详解](https://mp.weixin.qq.com/s/YxlAtPmCZ8h_oyPStXJP_A)
- [阿里终面：分布式事务原理](https://mp.weixin.qq.com/s/JZnLbBrRx_fDtnsYsRs4Aw)
- [分布式事务，阿里为什么钟爱TCC](https://mp.weixin.qq.com/s/eczKVv7Jgt4f0Mhwaq1JXw)
- [七种分布式事务的解决方案，一次讲给你听！](https://mp.weixin.qq.com/s/VIuJ5ywyjfGjAWd3Fb-XWg)
- [对比7种分布式事务方案，还是偏爱阿里开源的Seata，真香](https://mp.weixin.qq.com/s/J3BMnwRD-Ag8BAlJQiYuhg)
- [实战！阿里神器Seata实现TCC模式解决分布式事务](https://mp.weixin.qq.com/s/hBSY7VwHu9kM_3OrJ8WDwA)
- [分布式事务的6种解决方案，写得非常好！](https://mp.weixin.qq.com/s/Aj_BECgTZWkxX-dy5sayOw)
- [我还不懂什么是分布式事务](https://mp.weixin.qq.com/s/MbPRpBudXtdfl8o4hlqNlQ)
- [一文看懂分布式事务](https://mp.weixin.qq.com/s?__biz=Mzg2MDYzODI5Nw==&amp;mid=2247494401&amp;idx=1&amp;sn=915f97e20b4cb58bea2f638389ff60e5&amp;source=41#wechat_redirect)
- [面试官：聊聊分布式事务，再说说解决方案！](https://mp.weixin.qq.com/s/QpOwudYMY1HMRpU6SIXjzA)
- [看了那么多博客，还是不懂TCC，不妨看看这个案例！](https://mp.weixin.qq.com/s/83-I7hPDuWRTTfrldHJ0VA)
- [听说TCC不支持OpenFeign](https://mp.weixin.qq.com/s/EQuVJGFi6SEj3Qj2FS-uSg)
- [五分钟带你体验一把分布式事务](https://mp.weixin.qq.com/s/47efAPrm10l1Bxn1zECwvA)
- [XA事务水很深，小伙子我怕你把握不住](https://mp.weixin.qq.com/s/BJHmVkNrvNL87hBT8DM8vg)
- [你这Saga事务保“隔离性”吗？](https://mp.weixin.qq.com/s/cZabAt7JF4QrQHERHHAWjA)
- [哪种分布式事务处理方案效率最高](https://mp.weixin.qq.com/s/jcavJfjseBvaETAuTPnRqw)
- [一文搞明白分布式事务解决方案](https://mp.weixin.qq.com/s/6DOtO5OQyCL8bR03Z-3q9A)
- [手把手带领小伙伴们写一个分布式事务案例](https://mp.weixin.qq.com/s/fzlr-6pDPWKbwVuJlXe8sA)
- [Spring Boot多数据源如何处理事务](https://mp.weixin.qq.com/s/NbnCiRwRFUZGym5vDxOoPQ)
- [亿级流量架构分布式事务如何实现？](https://mp.weixin.qq.com/s/lwCNNCyG9wwtRHsni5pD8g)
- [SpringBoot分布式事务的解决方案（JTA+Atomic+多数据源）](https://mp.weixin.qq.com/s/ic57T3Yj2C_5tpdnM39IrQ)
- [分布式事务，原理简单，写起来全是坑！](https://mp.weixin.qq.com/s/29PmqK_bzDgh8bl9SBY3Uw)
- [分布式事务处理方案大PK！](https://mp.weixin.qq.com/s/kiRD3Hmdx2b__cBWeQOTWQ)
- [如何用RabbitMQ解决分布式事务](https://mp.weixin.qq.com/s/wTF3LlUKtH3lzsVgCLdCpQ)
- [MySQL为什么需要两阶段提交？](https://mp.weixin.qq.com/s/XRGIO7S9q9XqAfwqWr0OsQ)
- [阿里Seata新版本终于解决了TCC模式的幂等、悬挂和空回滚问题](https://mp.weixin.qq.com/s/nM81BRyQRTWab78a6KTD-g)




