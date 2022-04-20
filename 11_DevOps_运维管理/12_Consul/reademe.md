原文链接：https://blog.csdn.net/zhou920786312/article/details/115725644



一、术语
1.1、Agent
Agent是一个独立的程序，通过守护进程的方式，运行在consul集群中的每个节点上。
每个Consul agent维护它自己的服务集合以及检查注册和健康信息。
Agent根据节点的性质，分为Agent Server 和 Agent Client
所有的agent都有DNS或是HTTP接口
1.1.1、Agent Server
保存client的注册信息，保存集群的配置信息
负责参与Raft quorum，维护集群高可用
通过广域网与其他datacenter交换WAN gossip信息
将Agent Client请求转发到leader或是远程datacenter
1.1.2、Agent Client
Agent Client将 HTTP 和 DNS 接口请求转发给server。
Agent Client相对来说是无状态的，它的唯一后台活动是参与LAN gossip pool。
它的资源开销很小，只消耗少量的网络带宽。
1.2、数据中心
将一个数据中心定义为一个私有、低延迟和高带宽的网络环境。这不包括通过公共互联网的通信。
1.3、Gossip
consul是建立在serf之上的，它提供了一个完整的gossip协议，用在很多地方。
Serf提供了成员，故障检测和事件广播。
Gossip的节点到节点之间的通信使用了UDP协议。
1.4、LAN Gossip
指在同一局域网或数据中心的节点上的LAN Gossip池。

1.5、WAN Gossip
只包含server的WAN gossip ，这些服务器在不同的数据中心，通过网络进行通信。
1.6、RPC
这是一个请求/响应机制，允许client向server发送请求。

二、 服务发现和治理


2.1、服务注册
当服务Producer启动时，会将自己的信息(IP 和 Port)通过发送请求告知Consul，consul保存服务Producer信息

注册方式

HTTP API（8500 端口)
配置文件
2.2、服务发现
Consul 接收到 Producer 的注册信息后，每隔一段时间会向 Producer 发送一个健康检查的请求，检验Producer是否健康。
2.3、服务调用
当Consumer请求Product时，会先从Consul中拿到临时表，从临时表中选一个Producer信息(IP和Port)， 然后根据这个IP和Port，发送访问请求。

临时表(temp table)

只包含通过了健康检查的Producer信息
IP和Port等信息
每隔一段时间会更新
三、 通信接口
3.1、 RPC功能
用于内部通讯，如下

1. Gossip
2. 日志分发
3. 选主
1
2
3
3.2、HTTP API
1. 服务发现
2. 健康检查
3. KV存储等
4. 默认端口为8500
1
2
3
4
3.3、Consul Commands (CLI,命令行工具)
1. 可以与consul agent进行连接，提供部分consul的功能。
2. 本质是调用HTTP API
1
2
3.4、DNS
仅用于服务查询，例如

 dig @127.0.0.1 -p 8600 web.service.consul

 获取当前的服务"web"对应的可用节点
1
2
3
3.5、Consul 内部端口使用汇总


3.6、Consul 请求调用链路


四、 consul 去中心化思想实现
consul是基于Serf来实现的。
Consul使用serf提供的gossip协议来管理成员和广播消息到集群
4.1、 Serf
基于gossip协议来实现的
Serf是一个服务发现，编配工具，它去中心化，不像集中式结构那样统一分配管理。
Serf提供成员关系，纠错检查，广播等功能。
4.2、 gossip protocol
基于流行病传播方式,实现节点间信息交换，来确保网络中所有节点的数据一样。
节点间的交互方式主要以下有三种
4.2.1、Push
发起信息的节点 A 随机选择联系节点 B，并向B发送自己的信息，节点 B 在收到信息后，更新比自己新的数据
一般拥有新信息的节点才会作为发起节点。
4.2.2、Pull
发起信息的节点 A 随机选择联系节点 B，并从对方获取信息。
一般无新信息的节点才会作为发起节点。
4.2.3、Push&Pull
发起信息的节点 A 向选择的节点 B 发送信息，同时从对方获取数据，用于更新自己的本地数据。
五、 consul节点交互过程


5.1、选举leader
服务器 Server1、Server2、Server3 上分别部署了 Consul Server, 组成了Consule集群，通过raft选举算法, server2成为leader节点。

5.2、服务器调用Consul Client进行注册
服务器 Server4 和 Server5 上通过 Consul Client 分别注册 Service A、B、C

5.3、 Server节点信息同步
Consul Client将注册信息通过 RPC 转发到 Consul Server
信息保存在 Server 的各个节点中，并且通过 Raft 实现了强一致性。
5.4、服务调用
服务器 Server6 中 Program D 要访问 Service B

5.4.1、HTTP API 方式
Program D 先访问本机 Consul Client 提供的 HTTP API
Consul Client 会将请求转发到 Consul Server。
Consul Server 查询到 Service B 并返回给 Program D
Program D 拿到了 Service B 的所有部署的 IP 和端口，根据负载均衡策略，选择Service B 的其中一个并向其发起请求。
六、共识协议(Consensus Protocol)
基于raft协议

七、Consul中的Raft
7.1、术语
7.1.1、Peer set(对等集)
是参与日志复制的所有成员的集合
在Consul中，所有服务器节点都位于本地数据中心的对等集。
7.1.2、Quorum（法定人数）
Quorum是一个Peer set中的主要成员
对于一个大小为n的集合，Quorum要求至少有(n/2)+1个成员。
7.1.3、FSM
有限状态机。
FSM是有限状态和它们之间的转换的集合。在应用新日志时，允许FSM在状态之间进行转换。相同日志序列的应用必须导致相同的状态，意味着行为必须是确定的。
7.1.4、log entry
Raft系统中的主要工作单元
一致性的问题可以分解为日志备份。
log是一个有序的条目序列。如果所有的成员对条目内容和顺序意见一致，那我们就认为日志是一致的。
7.1.5、Committed Entry
当一个条目被持久地存储在节点的Quorum中时，它被认为是已经提交的。一旦提交了条目，就可以使用它。
7.1.6、Leader
在任何时刻，Peer set都会选举出一个节点来作为leader。
leader负责获取新的日志条目，复制到追随者。
7.2、Consul中的Raft
Consul中只有server节点会参与Raft算法并且作为peer set中的一部分。所有的client节点会转发请求道server节点。这个设计的部分原因如下

随着更多的成员被添加到对等集，quorum的大小也会增加。这将带来性能问题，因为您可能需要等待数百台机器对条目达成一致，而不是等待少数机器。
当启动时，单个Consul server的模式为bootstrap。

这个模式允许节点选举自身作为leader。一旦leader选举成功，其他服务器可以以保持一致性和安全性的方式添加到对等集。
一旦添加了几个服务器，就禁用bootstrap模式。
由于所有的server都处于对等集合中，它们都知道当前的leader是谁。当RPC到到一个非leader的server上时，请求会被转发到leader。如果PRC是一个只读的请求，leader会根据FSM的当前状态生成结果。如果RPC是一个修改状态的请求，leader会生成一个新的日志条目并使用Raft算法对它进行应用。一旦日志条目提交并且应用于FSM了，转换请求就完成了。

由于Raft的备份特性，它的性能对网络延迟是很敏感的。出于这个原因，每一个datacenter会选举独立的leader并且维护自己的不相交的对等集合。数据被datacenter进行分区，所以每个leader只负责处理它自己datacenter的数据。当请求被一个远程datacenter接收时，会被转发到正确的leader。

这种设计允许低延迟事务和高可用性，而不牺牲一致性。
7.3、Consul中的Raft注意点
只要有quorum个节点有效，Consensus是容错的。如果有效节点数无法达到的quorum，那么不可能处理log entry或维护成员关系。例如，假设只有2个peer：A和B。quorum也是2，这意味着这两个节点必须同意提交日志条目。如果A或B有一个失败，现在是不可能满足quorum个节点有效的条件。这意味着群集无法添加或删除节点或提交任何额外的日志条目。这样的结果就是集群不可用。这时，需要手动干预，删除A或B，然后以引导模式重新启动剩余节点。

Consistency Modes（一致性模式）
八、Consistency Modes（一致性模式）
Consul支持3种不同的读取一致性模式。分别如下

三种读取模式是：

8.1、default
Raft利用领导租赁，提供领导者认为其角色稳定的时间窗口。但是，如果领导者与剩余的同伴分开，则可以在旧领导者持有租约时选出新的领导者。这意味着有2个领导节点。由于旧的领导者无法提交新的日志，因此不存在裂脑的风险。但是，如果旧的领导者为任何读取提供服务，则这些值可能过时。
默认一致性模式仅依赖于领导者租赁，将客户端暴露给可能过时的值。我们做出这种权衡，因为读取速度快，通常强烈一致，并且只在难以触发的情况下过时。

由于分区导致领导者下台，因此陈旧读取的时间窗口也是有限的。

8.2、consistent
这种模式非常一致，没有任何警告。它要求领导者与法定人数确认它仍然是领导者。这为所有服务器节点引入了额外的往返。由于额外的往返，权衡总是一致读取但延迟增加。

8.3、stale
此模式允许任何服务器为读取服务，无论它是否是领导者。这意味着读取可以是任意陈旧的，但通常在领导者的50毫秒内。权衡是非常快速和可扩展的读取，但具有陈旧的值。

此模式允许在没有leader的情况下进行读取，这意味着无法使用的群集仍然可以响应。

九、Deployment Table（部署表）
3个节点的Raft集群可以容忍1个节点故障，5节点的集群可以容忍2个节点的故障。因此，Consul推荐部署包含3或5个Server节点的数据中心。这样可以最大限度地提高可用性，而不会大大牺牲性能。下面的表格总结了集群节点数目与容忍的故障节点数目：



十、Consul中的Gossip（流言传播协议）
Consul使用Gossip协议来管理成员和集群广播消息，这些都是通过使用Serf库的。
Serf所使用的Gossip协议基于SWIM（可伸缩的弱一致的感染模式的过程组成员协议），并做了一些细微的修改。
Consul用了两种不同的Gossip池。称为LAN池和WAN池。
10.1、LAN池
Consul中的每个数据中心有一个LAN池，它包含了这个数据中心的所有成员，包括clients和servers。
LAN池用于以下几个目的
成员关系信息允许client自动发现server, 减少了所需要的配置量。
分布式失败检测机制使得由整个集群来做失败检测这件事， 而不是集中到几台机器上。
gossip池使得类似领导人选举这样的事件变得可靠而且迅速。
10.2、WAN池
WAN池是全局唯一的，所有的server都应该在WAN池中，无论它位于哪个数据中心。
WAN池允许server跨datacenter请求
集成的故障检测功能使Consul能够优雅地处理整个datacenter失去连接或者或者仅仅是个别的数据中心的某一台失去了连接。
这些特性都是由Serf提供的，用户无需感知。