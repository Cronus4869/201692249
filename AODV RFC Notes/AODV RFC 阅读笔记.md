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
- Originator sequence number: 这个序列将用于指向源端的路由请求中
- Destination sequence number: 源端最近收到指向目的端的最新序列号



### RREQ

![RREP](./RREP.png)

- Type: 2
- R: Repair flag; used for multicast.
- A: Acknowledgment required.
- Prefix Size: 非0情况下这5bits表示下一跳可以被拥有相同前缀的节点使用
- Lifetime: 被RREP认为有效的时间



### RERR

![RERR](./RERR.PNG)

- Type: 3
- N: 本地修复，上游节点暂时不要删除
- DestCount: 消息中包含的不可达节点，至少为1
- Unreachable Destination Sequence Number：路由表项中之前的目的端不可达的目的端序列号



### RREP-ACK

- Type: 4



