+++
title= "【译】【原】【未完】kafka rebalance 协议详解"
date= 2020-11-22T16:05:52+08:00
categories = ["kafka"]
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
type = "post"
+++

> 原文：https://medium.com/streamthoughts/apache-kafka-rebalance-protocol-or-the-magic-behind-your-streams-applications-e94baf68e4f2

从Apache Kafka 2.3.0版本开始，其内部Rebalance协议，特别是用于Kafka Connector和consumer上的协议，已经有了多次重大改进，这个协议现在看起来变复杂了很多，而且有时候看起来很炫酷。

这篇文章会始于rebalance protocol的基础，即kafka consumer协议的核心，随后，我们会讨论他的限制以及最新的改进。

# Kafka & The Rebalance Protocol 101

## Let’s go back to some basics
Apache Kafka是一个基于分布式、发布/订阅模式的流式处理平台。首先，producers进程发送消息到topics，broker集群管理并且保存这些topic。消费者进程订阅这些topic来获取并处理发布其上的消息。一个topic分布于多个broker，即每个broker管理一个发布到该topic的消息的子集，而这个子集被称为partition。topic创建时确定partition的数量，并且这个数量可能随时间而增长（要留意这个操作带来的影响）。partition的概念非常重要，因为他是Kafka的producer和consumer进程执行并行操作的最小单元。

partitions允许producer进程并行写入消息。消息发送时可指定一个key，该key被用来进行hash从而计算出消息该被发送到哪个目标partition。这个机制确保了发送时指定了相同key的消息都会被发送到相同的partition。此外，该机制同时确保了consumer进程会按照发送的顺序消费该partition上的消息。

partition的数量同时决定了某一个consumer group中活跃consumer进程的最大数量。consumer group是kafka为了对多个consumer client分组而提出的一个逻辑组机制，目的是使多个partition被负载均衡地消费。Kafka确保同一个topic下的一个partition只能同时分派给consumer group中的一个consumer。

下图说明了这个概念，A是一个consumer group，该group下有3个consumer。这些consumer订阅了topic A，partition分配如下：P0->C1, P1->C2, P2->C3, P3->C1。

![image](/img/blog/kafka-rebalance-protocol/Consumer-Group.jpeg "Apache Kafka — Consumer Group")

如果某个consumer由于被手动停掉或者挂掉，而离开了这个组，则partition会在该组剩下的consumer中进行自动重新分配。同样的，如果一个consumer(重新)加入到这个组，也会触发相同行为。

Kafka Rebalance Protocol即被用于这种多个consumer客户端在一个动态组中协作的场景

让我们深挖一下这个协议

## The Rebalance Protocol, in a Nutshell

首先，我们先定义何为Apache Kafka的Rebalance。

Rebalance/Rebalancing: 一个流程，即多个使用kafka客户端和（或）Kafka coordinator的分布式进程，构成了一个通常意义上的组，并且使一组资源分布于该组的成员进程上（原文：https://cwiki.apache.org/confluence/display/KAFKA/Incremental+Cooperative+Rebalancing%3A+Support+and+Policies）

以上定义并没有引用任何有关consumer或partition的概念，而是使用了成员和资源的概念，是因为Rebalance Protocol不只是能管理consumer，同时也可用于协调任何进程组。

这里是一些Rebalance Protocol的用法：

* Confluence Schema Registry 依赖rebalancing选主。

* Kafka Connect 用其在多个worker中分配task和connector。

* Kafka Stream 用其对多个应用流实例分配task和partition。

![image](/img/blog/kafka-rebalance-protocol/Rebalance-Protocol-components.jpeg "Apache Kafka Rebalance Protocol and components")

此外，Rebalance机制包含两个重要协议（划重点）：Group Membership Protocol和Client Embedded Protocol。

首先说Group Membership Protocol，就像这个名字一样，这个协议被用来管理组内成员的协作。当一个客户端要加入一个组时，他要和Kafka broker（也即coordinator）之间产生一系列的请求/响应。

Client Embedded Protocol是执行于客户端侧的协议，他通过嵌入第一个客户端，从而扩展该客户端的功能。比如这个协议被用于partition在组成员间的分配。

现在我们对Rebalance Protocol有了更深一步的认识，接下来让我们演示一下他是如何应用于在consumer组内分配partition的

## JoinGroup
当consumer启动时，它会发送一个FindCoordinator请求获取负责协调所在消费组的Kafka broker coordinator，随后便通过发送JoinGroup请求初始化Rebalance protocol

![image](/img/blog/kafka-rebalance-protocol/JoinGroup-Request.jpeg "Consumer — Rebalance Protocol — JoinGroup Request")

可以看到，JoinGroup请求包含了一些consumer端的配置，如session.timeout.ms和max.poll.interval.ms，这些属性被coordinator用于在得不到成员回应时，把该成员移出组的逻辑。

此外，这个请求同时包含了两个非常重要的字段：成员支持的protocol列表，和被用于执行某个embedded client protocol的元数据。在这个例子里，成员协议列表是由partition的分配者为consumer配置的（比如，partition.assignment.strategy），元数据包含了consumer订阅了的topic列表。

如果你不知道这些数据的功能，推荐你阅读官方文档。

JoinGroup行为扮演着一个屏障的角色，即如果没收到consumer发送的请求（比如，在group.initial.rebalance.delay.ms限定的时间内），或者达到了Rebalance超时时间，那么coordinator就不会发送任何响应了。

![image](/img/blog/kafka-rebalance-protocol/JoinGroup-Response.jpeg "Consumer — Rebalance Protocol — JoinGroup Response")

组内第一个consumer会成为当前组的leader，收到当前consumer组的活跃成员列表，并且选择一种分配策略，其他consumer则只会收到一个空的响应。组leader负责执行本地partition的分配工作。

## SyncGroup
之后，所有成员都发送给coordinator一个内容为空的SyncGroup请求，而组leader会在该请求中附上其计算过的分配方案。

![image](/img/blog/kafka-rebalance-protocol/SyncGroup-Request.jpeg "Consumer — Rebalance Protocol — SyncGroup Request")

当coordinator响应了所有的SyncGroup请求，每个consumer会收到分配与其的partition，依据配置的listener执行PartitionAssignmentMethod方法（译注：这里我也没太懂），并开始收取消息。

![image](/img/blog/kafka-rebalance-protocol/SyncGroup-Response.jpeg "Consumer — Rebalance Protocol — SyncGroup Response")

## Heartbeat

最后重要一点，每个consumer会定期（heartbeat.interval.ms）向brokercoordinator发送hearbeat请求，从而保持session活跃。

如果一个Rebalance任务开始执行了，则coordinator会使用heartbeat的响应，去告知所有consumer重新加入这个组

![image](/img/blog/kafka-rebalance-protocol/Heartbeat.jpeg "Consumer — Rebalance Protocol — Heartbeat")

在真实的分布式场景或者任何场景中，错误总会发生，硬件会损坏，网络或者consumer可能会发生瞬时的错误。然而所有这些错误都会触发Rebalance。

## Some caveats
首先，Rebalance protocol为了处理一个成员，必须要停止组内所有成员（STW效应）

比如，让我们正常地停掉一个组内实例，则会发生第一个Rebalance场景，这个被停掉的consumer会在停止前，发送给coordinator一个LeaveGroup请求

![image](/img/blog/kafka-rebalance-protocol/LeaveGroup.jpeg "Consumer — Rebalance Protocol — LeaveGroup")

剩下的consumers会在下一次心跳时，被通知要执行Rebalance流程了，所以要重新执行一次JoinGroup/SyncGroup流程，从而去获取重新分配的partition列表

![image](/img/blog/kafka-rebalance-protocol/Rejoin.jpeg "Consumer — Rebalance Protocol — Rejoin")

在整个Rebalance过程中（partition被重新分配前），consumers不再处理任何数据。并且默认Rebalance超时时间被固定为5分钟，在不断增长的consumer时间延迟的场景中，这么长的超时时间会造成很大问题。

就会发生这样的问题，比如，consumer刚从一个瞬时的错误中恢复，则他重新回到这个组时，就会触发一次Rebalance并且导致所有consumer停止（而在错误发生时已经STW了一次）

![image](/img/blog/kafka-rebalance-protocol/Restart.jpeg "Consumer — Rebalance Protocol — Restart")

另外一个导致重启的原因是全组成员滚动升级，这对consumption group是一个十分严重情况。对一个有三个consumer的组，这种操作会触发6次Rebalance，从而严重影响到消息处理过程

最后要说，在Java环境下运行的kafka consumer通常会有这两个常见问题：
1. 丢失heartbeat请求 -- 可能由于网络超时，gc暂停
2. 没有定期执行KafkaConsumer#poll()方法 -- 由于一个长时间的消息处理
在问题1的场景中，coordinator在session.timeout.ms配置的时间内，没有收到heartbeat请求，认为这个consumer已经挂掉了。
在问题2的场景中，处理拉取到的消息的时间超过了max.poll.inteval.ms配置的时间。

Image for post
Consumer — Rebalance Protocol — Timeout

# Static Membership
为了减少由于consumer发生瞬时失败导致的Rebalance，Apache Kafka 2.3版本引入了Static Membership的概念（KIP-345）

The main idea behind static membership is that each consumer instance is attached to a unique identifier configured with group.instance.id. The membership protocol has been extended so that ids are propagated to the broker coordinator through the JoinGroup request.
static membership的主要方案是每个consumer绑定一个唯一标识id（配置于group.instance.id），并且这些consumer的id会在发送JoinGroup请求时发送给coordinator -- 依据扩展后Membership protocol

Image for post


If a consumer is restarted or killed due to a transient failure, the broker coordinator will not inform other consumers that a rebalance is necessary until session.timeout.msis reached. One reason for that is that consumers will not sendLeaveGroup request when they are stopped.
如果一个consumer重启或者由于瞬时错误而被kill了，因为这个consumer不是通过正常发送LeaveGroup请求而停止的，则在触达session.timeout.ms时间以前，broker coordinator不会通知其他consumer进行Rebalance流程

Image for post

When the consumer will finally rejoin the group, the broker coordinator will return the cached assignment back to it, without doing any rebalance.
当这个consumer从错误恢复，并重新回到组时，broker coordinator会把之前缓存了的任务重新发给他，而不用经历任何Rebalance过程

Image for post

When using static membership, it’s recommended to increase the consumer propertysession.timeout.mslarge enough so that the broker coordinator will not trigger rebalance too frequently.
当使用static membership时，建议加大consumer的session.timeout.ms的属性值，以防broker coordinator会频繁触发Rebalance。

On the one hand, static membership can be very useful for limiting the number of undesirable rebalances and thus minimizing stop-the-world effect. On the other hand, this has the disadvantage of increasing the unavailability of partitions because the coordinating broker may only detect a failing consumer after a few minutes (depending onsession.timeout.ms). Unfortunately, this is the eternal trade-off between availability and fault-tolerance you have to make in a distributed system.
一方面说，static membership机制可以有效避免无效的Rebalance从而最少化STW影响。但从另一方面，这也降低了partition的可用性，因为coordinator可能要数分钟后（session.timeout.ms）才能检测到一个挂掉的consumer。然而，这即是我们在分布式系统中永恒的话题：可用性VS分区容错性（译注：不应该是可用性和一致性么？）


# Incremental Cooperative Rebalancing
As of version 2.3, Apache Kafka also introduces new embedded protocols to improve the resource availability of each member while minimizing stop-the-world effect.
除了减少STW影响的改进，在Apache Kafka 2.3版中还引入了一些新的embedded protocol来增强资源可用性。

The basic idea behind these new protocols, is to perform rebalancing incrementally and in cooperation — In other words, it means executing multiple rebalance rounds rather than a global one.
这些新增protocol的主要想法是：增量和协作地Rebalance，换句话说，这也意味着要执行多次Rebalance

Incremental Cooperative Rebalancing was first implemented for Kafka Connect through the KIP-415 (partially implemented in Kafka 2.3). Moreover, it will be available for streams and consumers from Kafka 2.4 through the KIP-429.
增量协作地Rebalance最开始被Kafka Connect引入（KIP-415，Kafka2.3也部分实现了）。并会在Kafka 2.4版本中对stream和consumer可用（KIP-429）

## Kafka Connect Limitations
Kafka Connect uses the group membership protocol to distribute connectors and tasks evenly among workers that compose a connect cluster. Thus, workers coordinate each other to rebalance connectors and tasks when a node fails/restart, tasks scales up/down and when configuration is submitted/updated.
However, before Kafka 2.3, whenever one of these scenarios occurred, the execution of all existing connectors was interrupted (i.e stop-the-word). Therefore, it was difficult to scale up a mutualized cluster with several dozens of connectors.
The Incremental Cooperative Rebalancing attempts to solve this problem in two ways :
1 ) only stop tasks/members for revoked resources.
2 ) handle temporary imbalances in resource distribution among members, either immediately or deferred (useful for rolling restart).
For doing that, the Incremental Cooperative Rebalancing principal is actually declined into three concrete designs:
Design I: Simple Cooperative Rebalancing
Design II: Deferred Resolution of Imbalance
Design III: Incremental Resolution of Imbalance
To give you a better understanding on how Incremental Cooperative Rebalancing works we are going to illustrate the design II in the context of Kafka Connect.
Deferred Resolution of Imbalance
First, let’s start with a simple connect cluster compose of three workers with this initial task/connector assignment :
Image for post
1 — Initial assignment
Now, let’s imagine that W2 fails without any particular reason and leaves the group by session timeout. A rebalance is triggered and remaining workers W1 and W3 rejoin the group. While sending a JoinGroup request workers include their previous assignment. Assignments are shared using the existing field member_metadataof the Group Membership protocol.
Image for post
2 — W2 leaves the group and rebalance is triggered (W1, W3 join).
W1 is elected as the group leader and performs tasks/connectors assignments by computing the difference with the previous assignments. Here, the leader detects that some task and connector are not presented in previous assignments.
Image for post
3 — W1 becomes leader and computes assignments
W1 send the new assigned tasks/connectors as well as revoked. You can note that W1 will not actually try to resolve immediately missing assignment (or imbalance). Instead of that, it will deferred the resolution by scheduling a next rebalance to get a chance to the failing member to reappear. The scheduling delay is fixed by a new configuration scheduled.rebalance.max.delay.ms(by default, it is equal to 5 minutes).
Note : With Incremental Cooperative Rebalancing, when a member receives a new assignment, it will start processing any new partitions (or tasks/connector). Moreover, if the assignment also contains revoked partitions then it stops processing, commit and then initiate another join group immediately. This has the effect of increasing the number of rebalancing but only stopping the resources whose assignment has changed.
Image for post
4 — W1, W3 receive assignments
W2 rejoins the group before delay expires and another rebalance is triggered. W1 and W2 also rejoin the group.
Image for post
5 — B rejoins the group before delay expire and a rebalance is triggered
However, W1 will not reassign missing task/connector until the scheduled rebalance delay expires.
Image for post
6 — W1 becomes leader and computes assignments
After the remaining delay expires, a final rebalance is triggered and all workers rejoin the group.
Image for post
7 — W1, W2, W3 receive assignments
Finally, the group leader reassigned A-Task-1 and Connector-B to W2. During all the rebalance sequence, W1 and W3 never stopped their assigned tasks.
Image for post
8 — After delay, all members join
Conclusion
The rebalancing protocol is an essential component of the consumption mechanism in Apache Kafka. But, it also serves as a generic protocol for coordinating group members and distributing resources among them (e.g Kafka Connect). Static Membership and Incremental Cooperative Rebalancing are both important features which provides a huge improvement to Apache Kafka by making this protocol more robust and scalable.


To learn more about the rebalance protocol and how it works have a look to the following links.
Sources :
KIP-429, KIP-415, KIP-453
https://cwiki.apache.org/confluence/display/KAFKA/Incremental+Cooperative+Rebalancing%3A+Support+and+Policies
https://kafka.apache.org/protocol
The Magical Rebalance Protocol of Apache Kafka by Gwen Shapira
https://www.slideshare.net/ConfluentInc/everything-you-always-wanted-to-know-about-kafkas-rebalance-protocol-but-were-afraid-to-ask-matthias-j-sax-confluent-kafka-summit-london-2019
https://www.slideshare.net/ConfluentInc/rebalance-protocol-insideout-a-developer-perspective
