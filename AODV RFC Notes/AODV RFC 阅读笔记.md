# AODV RFC 阅读笔记

## Feature

- 能快速获取新目的地路由
- 无须维护到未处于活跃通信中的节点的路由
- 连接中断时，通过通知受影响的节点集合来让使用了断开的链接的路由时效
- 每条路由表项使用一个目的端序列
- 目的端序列号由目的节点和目的节点发送给请求节点的路由信息共同生成
- 目的端序列号避免了环的产生，易于实现
- 请求节点需要选择最好的序列号
- 基于UDP，port：654



## Principle

- 消息类型：RREQ，RREP，REER
- 通信两端的有效路由互相包括对方时，AODV没有任何作用
- 需要新目的端的路由时，广播RREP找到目的地路线，当RREQ到达目的地或到达具有到目的地Fresh enough的中间节点，可以确定路线。通过RREP单播回到源端可以获得路由，接收请求的每个节点缓存返回给源端的路由，由此单播返回
- 检测到活跃路由中的链路断开时，用RERR通知其它节点该链路丢失了，RERR消息指出了由于已断链路不再可达的目的地或是子网，为了实现这种机制，每个节点需要包含它的前项节点，以及它的可能用到作为下一条的邻居节点的IP，该消息可以通过RREP的迭代得到。如果RREP有非零前缀长度，那么源端也包含在子网的前项列表里。
- 序列号由起始节点主导



## Message format

### RREQ

![RREQ](./RREQ.png)

- Type: 1
- J: Join flag; reserved for multicast.
- R: Repair flag; reserved for multicast.
- G:Gratuitous RREP flag.
- D: Destination only flag.
- U: Unknown sequence number.
- Sequence number: RREQ ID，用序列和IP唯一标识源端
- **Originator sequence number**: 这个序列将用于路由表项中，指向路由请求的发起端
- **Destination sequence numbe**r: 源端过去收到的指向目的端的最新序列号



### RREP

![RREP](./RREP.png)

- Type: 2
- R: Repair flag; used for multicast.
- A: Acknowledgment required.
- Prefix Size: 非0情况下这5bits表示下一跳可以被拥有相同前缀的节点使用
- Lifetime: 被RREP认为有效的时间



### RERR

![RERR](./RERR.png)

- Type: 3
- N: 本地修复，上游节点暂时不要删除
- DestCount: 消息中包含的不可达节点，至少为1
- Unreachable Destination Sequence Number：路由表项中之前的目的端不可达的目的端序列号



### RREP-ACK

- Type: 4



## Operation

### 管理序列号

每个节点的路由表项都必须包括最新的目的地序列号(包括IP消息)，当一个节点从RREQ，RREP，RERR收到关于目的地的新的消息时，目的端序列号更新。AODV通过目的端序列号来避免成环。**以下情况目的端节点会更新它的序列号**：

- 当一个节点发起路由发现时，它必须增加序列号。这避免了与先前发起RREQ的源端建立反向路由时的冲突。
- 当一个节点对RREQ发起RREP回复时，它必须将它的序列号更新为RREQ里的序列号和目的地序列号中的最大值。

序列号大小的比较：为了防止翻转带来的影响，比较时将序列号视为32位无符号整数，节点用从AODV消息中收到的序列号减去当前序列号，如果差值小于0，则丢弃AODV消息中与目的端相关的信息，因为它不是新的。

唯一一种例外状况是，一个节点可能改变路由表中的某条表项的目的端序列号来响应到达目的端的下一跳丢失或是失效。节点通过查询路由表中的特定下一跳来查找目的端。在这种情况下，每个用到丢失的这一跳的目的端节点会增加序列号，并将路由设为不可用。当包含受影响的目的端的足够新鲜的路由消息被标记了路由项不可用的节点收到时，节点会据此更新路由消息。**节点在以下情况会改变目的端相关的路由表项的序列号**：

- 自己是目的节点，并能提供到自己的新的路由
- 收到关于目的端的新的序列号的AODV消息
- 到达目的端的路径断了或是到期了



### 路由表和前项列表

当一个节点收到来自邻居的AODV控制消息，或是新建或更新了到达某一特定目的端或是子网的路由时，它会检查路由表中的到那个目的端的表项。如果没有目的端的表项，就创建一个。序列号由包含了control packet的消息决定，或是将Sequence Number Field设为false。路由仅在以下情况下更新：

- 新的序列号比路由表中的目的端序列号高
- 序列号相等，但是跳数+1比路由表中的存在的跳数多
- 序列号未知

路由表项的Lifetime要么由Control packet决定，要么初始化为ACTIVE_ROUTE_TOMEOUT, 这个路由现在可以用于发送任何包和路由消息。每次使用路由转发数据包时，路径的源端、目的端、下一跳的Active Route Lifetime域会被更新为大于等于当前时间加上ACTIVE_ROUTE_TOMEOUT。由于源端和目的端之间的路由应是对称的，前一跳的Active Route Lifetime以及指向源端IP的反向路径也应该作出这样的更新。无论是目的端是一个子网还是单一节点，当使用路由时，Active Route的Lifetime都会更新。每个节点维持有效路由的路由表项，同时也维持一个前项列表用于在路由上转发packet。当节点侦查到下一跳丢失的时候，这些前项会收到通知。路由表项的前项列表包含了发起或是转发路由回复的邻居节点。



### 发起路由请求

当一个节点确定了自己需要一条到达目的端的路由但是还没有可行的路由的时候，它会传播一个RREQ消息。具体来说就是这种情况，如果一个目的端之前对于一个节点来说是未知的，或者之前到达目的端的有效路由到期或者是失效了。RREQ消息的目的端序列号是最新知道的目的端序列号，它是从路由表的目的端序列号域复制过来的。如果没有已知的序列号，就将Unknown Sequence Number标志置1。RREQ消息的源序列号是节点自己的序列号，它在插入一个RREQ之前或自动增加。RREQ ID域在上一个在当前节点使用的上一个RREQ ID之上+1，每个节点都只有一个RREQ ID，Hop Count 域置零。

在广播RREQ之前，源节点缓存RREQ ID和它自己的IP地址用于PATH_DISCOVERY_TIME，这样，当一个节点多次收到来自邻居的同一个包时，它就不会重复处理和转发。

源节点被期待与目的节点能有双向通信。为完成这个目标，任何中间节点的生成用来传递给源节点的RREP必须被一些action完成，即告诉目的节点一条回到源节点的路由。具体实现是通过设置G flag。

一个节点每秒发送的RREQ不能超过RREQ_RATELIMIT条，在广播RREQ之后，节点等待RREP或者是其他带有关于目的节点的路由的当前控制消息。如果在NET_TRAVERSAL_TIME毫秒内节点没收到任何消息，节点可能会再次尝试广播RREQ来发现路由，这取决于在最大TTL值下的RREQ_RETRIES次数的上限。每次新的尝试都应该增加并更新RREQ ID，每次尝试IP首部的TTL field根据特定机制来设置(下文详述)，这是为了能控制每次重发中RREQ的转发距离。

等待路由的数据包应该被缓存(例如发送出一个RREQ之后等待RREP)，该缓存为FIFO。如果超过最多尝试次数都没有收到，所有和目的端相关的数据包都应该被丢弃，目的端不可达消息会被传送。

为了解决网络拥堵，源端发起的对单个目的节点的路由发现尝试必须使用用二进制指数回退方案，即等待时间逐次乘2。

### 路由请求消息的传播控制

为了避免不必要的RREQ在网络范围内的传播，源端必须使用扩展环搜索技术，即源端在RREQ包的IP head初始化TTL = TTL_START，设置接收RREP的timeout为RING_TRAVERSAL_TIME，用于计算RING_TRAVERSAL_TIME的TTL_VALUE设为与IP首部的TTL field的值相等。如果RREQ超时，将TTL加上TTL_INCREMENT重新广播RREQ, 直到RREQ的TTL设置到达TTL_THRESHOLD，超过的时候就使用TTL = NET_DIAMETER，每次timeout都为RING_TRAVERSAL_TIME。当所有重试需要遍历整个ad hoc网络时，可以通过配置TTL_START和TTL_INCREMENT与NET_DIAMETER相等来实现。

存储在无效路由表条目中的Hop Count表示最后一次知道的路由表中到目的端的跳数，如果之后需要到达这个目的端的新的路由，RREQ的IP首部TTL会被初始化为Hop Count + TTL_INCREMENT。然后每次超时TTL都会增长TTL_INCREMENT直到TTL到达TTL_THRESHOLD。超过之后TTL会被设为NET_DIAMETER，timeout设为NET_TRAVERSAL_TIME。

到期路由表项在current_time + DELETE_PERIOD之前不应被立即抹去，否则，与路由有关的soft state会丢失，另外，长路由表项抹杀时间也许会被配置，任何等待RREP的路由表项在(current_time + 2 * NET_TRAVERSAL_TIME)前不应被抹去。



### 处理和转发路由请求

当一个节点收到RREQ，它首先新建或是更新到前一跳的路由，这个路由不带有效序列号；然后检查它是否曾在上一个PATH_DISCOVERY_TIME内收到了有着同样源IP地址和RREQ ID的RREQ。如果有收到，丢弃新收到的RREQ。对于没被丢弃的RREQ：

1. RREQ的hop count加一；
2. 节点用最长前缀匹配搜索到达源IP的反向路由。如果需要，用路由表中RREQ的源序列号创建或是更新路由。
3. 当节点收到要返回给发起RREQ的节点的RREP，反向路由就会被需要。

当反向路由创建或是更新，以下路由动作会被执行：

- RREQ的源端序列号与路由表项中的相关目的端序列号进行比较，如果大于现有值就复制；
- valid sequence number field 设为真
- 路由表中的下一跳设为发送RREQ的节点(由IP首部中的源IP地址得到，一般不等于RREQ中的源IP地址)
- hop count从RREQ消息中的Hop Count字段复制过来

当RREQ被接收，源IP地址的反向路由表项的Lifetime被设置为max(ExistingLifetime, MinimalLifetime)，当前节点可以使用反向路由转发数据包。

如果一个节点并没有发起RREP并且收到的IP header的TTL大于1，节点更新并广播RREQ到每个配置接口上。为了更新RREQ，发出的IP首部的TTL或者hop limit应该减1，为了通过中间节点计算新的一跳。最后，目的端序列号设为收到的RREQ消息中有关值的最大值。目的端序列号仅由请求目的端的节点维持，转发节点不能修改目的端序列号，即使收到的RREQ的值比当前当前转达节点中的值大。

否则，如果一个节点并没有发起RREP，那么节点转发RREQ，*需要注意的是，中间节点如果向每一个RREQ的转发回复一个特定的目的端，它可能会变成目的端没有收到任何发现消息。*这种情况下，目的端没有从RREQ消息中学习到到达源端的任何路由，可能会导致目的端发起路由发现。为了目的端能够学习到到达源端的路由，在目的端可能需要到达源端的路由时，无论出于什么原因，源端节点都需要在RREQ里设置G flag。为响应设置了G flag的RREQ，中间节点会返回RREP，并单播一个无目的的RREP给目的端节点。



### 发起路由回复

节点在以下情况下发起RREP:

- 自己是目的端
- 有到达目的端的活跃路由，在节点的现有路由表中目的端序列号有效并且比RREQ的目的端序列号要大，同时destination only（D）flag未被设置。

当发起路由回复时，节点从RREQ复制目的端IP和源端序列号到RREP的相关域里。处理过程根据节点是被请求的目的端还是拥有足够新鲜的到达目的端路由的中间节点而有所不同。只要被创建，RREP就通过单播到下一跳就是RREQ源端的节点，就像被发起者路由表项指出的的那样。hop counter指出了这个过程的距离。

#### 目的端的路由回复

