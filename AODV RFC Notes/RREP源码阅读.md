# RREP源码阅读

aodv_rrep.h中主要定义了7个函数：

- 创建RREP
- 创建RREP-ACK
- 发送RREP
- 转发RREP
- RREP的处理
- RREP-ACK的处理

## RREP消息结构

正如rfc里所提及的RREP格式：

![RREP](./img/RREP.png)

RREP结构体被定义为这样的一个结构体：

![struct_rrep](./img/struct_rrep.png)

详细描述参见AODC RFC阅读笔记的RREP部分。

##  创建RREP

定位：aodv_rrep.c

```
52~54: 创建一个 AODV_msg 消息并返回指针，强制转换成RREP *类型。我们来看一下aodv_socket 消息是如何创建的：
```

![aodv_socket_new_msg](./img/aodv_socket_new_msg.png)

如上图，它是通过调用系统 `memset` 方法，将 `sendbuf` 全部初始化为`\0`，然后强制转换成 `AODV_msg` 消息。

```
55~68: 将RREP结构体的成员进行初始化，值得注意的是，这个函数接收的参数 flags 用来处理 repair 标志和 ack 标志，通过位与。
```

```
72~75: 进行DEBUG输出。
```



## 创建RREP-ACK

RREP-ACK的创建和RREP十分类似，都是创建一个原始的 AODV_msg 然后转换成其指针类型。需要注意的是，RREP-ACK仅仅是为了确定当前连接是双向的，因此，它仅需要包含一个type信息（最多还有存储信息）。

定位：aodv_rrep.c

```
84~87: 声明 RREP-ack 指针；创建 aodv_msg ，返回指针并强制转换成 RREP-ack类型；将结构体的type成员变更为ack类型。
89：DEBUG日志
```



## 处理RREP-ACK

RREP-ACK的处理涉及到路由表，我们先来简单了解一下路由表项在实现中的结构：

![route_table_entries](./img/route_table_entries.png)

`list_t` 是一个包含了前项指针和后项指针的简单结构体；此外路由表项包含了诸多的条目：目的端，目的端序列号，下一跳，跳数，路由标志，表项状态，多种定时器，前项总数，前项列表等，如上所示。

接着来看看路由表：

![routing_table](./img/routing_table.png)

如上所示，路由表记录表项条目，活跃路由数，并用一个 ` list ` 数组来记录所有项的前项和后项。

定位：aodv_rrep.c

```
98~100: 声明一个 rt_table_t 指针，这里的 rt_table_t 其实就定义为了 rt_table；调用 rt_table_find 函数查找路由表中是否存在和源端ip一致的路由表项，并返回赋值给 rt_table_t 指针。
102~106: 如果没有找到，指针为空，那么在相应警告日志中记录下应该回传ACK的那个节点的ip，程序返回。
107: 如果找到了，在相应DEBUG日志中记录回传ack的节点ip
110: 移除没有超时的timer	// 为什么要移除呢？
```

> 移除定时器的原因将在 time_queue 部分的阅读中解释= =



## RREP消息扩展

源码还提供了对超出 `RREP_SIZE` 的 `rrep` 的扩展服务。本文不认为它是重点，随便解释一下：

```
116~119: 显然，小于最大 SIZE 的消息是不需要扩展的。
121: 用原来 rrep 部分和超出的 offset 部分相加得到扩展后的部分
123~124：给 AODV_ext 结构体变量赋值
126: 将参数 data 复制到扩展后的结构中
```



## RREP的发送

发送 `RREP` 的时候要做三件事：

- 查看反向路由表
- 检查是否需要ACK
- 更新前项列表

函数定义如下

```c
void NS_CLASS rrep_send(RREP * rrep, rt_table_t * rev_rt,
                         rt_table_t * fwd_rt, int size);
```

如上，它接收RREP指针，反向路由表项和转发路由表项，size参数，进行发送处理。

### 查看反向路由表：

```
137~140: 如果反向路由表不存在，将警告消息写入 warning 日志，程序结束，因为此时不能发送 RREP。
142: 设置in_addr类型变量dest的源地址为RREP中的目的端地址。
```



### 检查是否需要请求ack：

```
145~146: 如果反向路由表项的状态有效并且这个表项被标记为单播，或者是反向路由表中的跳数为1并且unidir_hack的值变成了非零值，那我们就可能需要请求ack，不过我们先再确认一下
147: 调用rt_table_find()找到路由表中目的为反向路由表项的下一跳的表项，存起来，这就是到邻居的表项。
```

这里可能需要说明一下 `rt_table_find()` 这个函数，我们先来看一下它的调用：

```c
rt_table_t *neighbor = rt_table_find(rev_rt->next_hop);
```

然后看一下它是怎么实现的：

![rt_table_find](./img/rt_table_find.png)

详细解释会放在 `routing_table.c` 的源码阅读里，总之，调用实现的功能是在全局变量 `rt_tbl` 里找到目的地为反向路由表项的下一跳的表项，并返回指针。

```
149: 如果到邻居的表项存在并且有效，并且表项的ack计时器还没被用过，这说明我们从这个邻居节点收到过RREQ，这条路径有可能是单向的，最好请求一下ack
153~154: 将到邻居的表项的标记为单播类型
159: 设置表项的单播标志之后要移除邻居节点的hello计时器，否则在忽略hello以后路由可能会超时
160: 在这种可能的单向链路中我们认为到邻居的路径断了，交给 neighbor_link_break() 进行rerr 处理。
162~163: 将单播链路写入debug日志
165: 将邻居的ack计时器的超时时间设置为NODE_TRAVERSAL_TIME，具体设置看下图，或是param.c
```

![NODE_TRAVERSAL_TIME](./img/NODE_TRAVERSAL_TIME.png)

> warning：160行的注释可能不对，等读过 rerr 部分源码之后再回来修改=、=
>
> 153：变量rrep_flags没有任何作用？

```
169~175: 将发送rrep给下一跳简单写入debug日志，调用aodv_socket_send发送给下一跳
```



### 更新前项列表

```
177~180: 更新转发路由表和反向路由表
```

> Tips：routing_table.h 里面有一个全局变量rt_tbl，它是 `routing_table类型` ，用来存储邻居节点， `rt_table_find ` 就是使用了这个全局变量来找到到达反向列表的下一跳的表项；而 `fwd_rt` 以及 `rev_rt` 都是 `rt_table_t` 类型，也就是说他们只是表项而已，虽说是表项，但是却实际也存储着前项节点。

```
182~184：在一定条件下，重新开启hello
```



## 转发RREP

函数定义如下：

```c
void NS_CLASS rrep_forward(RREP * rrep, int size, rt_table_t * rev_rt, 						rt_table_t * fwd_rt, int ttl);
```

如上，它接收RREP指针，size 和 ttl，以及转发和反向路由表项，进行转发处理。

```
190~198: 异常情况处理，当没有转发表或者反向表的时候，记录异常并返回；当没有RREP指针为空的时候，记录异常并返回。
200: 写入正常日志：“我把rrep转发给了下一跳xxx”
204: 作者想要检查需要请求RREP_ACK的条件，但是他也不知道怎么确定条件来怀疑链路单向，就写了一段永不执行的代码=、=不过我们还是可以来看看他的想法
210~213: 找到的rrep源的下一跳
215~223: 设置rrep的A flag为1，设置到达邻居的表项为单向链路类型，设置ack计时器的超时时间
```

> Tip：不用担心 rrep 的 a flag 没被设置，创建 rrep 的时候 传入的 flags 参数会引导设置 a flag。

接下来是代码真正被执行的部分：

```
226: 调用 aodv_socket_queue_msg，把 aodv 消息复制到 send buffer 里面，返回buffer指针
227: 更新hopcount
228: 将消息转发给下一跳
229: 更新转发表项和反向表项的前项列表
235: 更新反向路由表项的超时时间
```



## RREP处理

```c
void NS_CLASS rrep_process(RREP * rrep, int rreplen, struct in_addr ip_src,
                            struct in_addr ip_dst, int ip_ttl,
                            unsigned int ifindex);
```

如上，它接收RREP消息指针，消息长度，源和目的地址，ttl，以及网口号ifindex，进行处理。

### 预处理

```
243~252: 声明相关变量
256~259: 将rrep的源和目的地址、目的端序列号、lifetime分别赋值给相关变量
261: hopcount加1
263~268: 如果rreplen的长度小于RREP的大小，说明传送过程发生了异常，写入日志，函数返回
271~272: 忽略自己发送到自己的rrep消息
274~278: 写入DEBUG日志，如果需要输出DEBUG消息
```



### 处理扩展部分

```
283~312: 长度大于RREP默认长度则进入循环，找到扩展部分，进行处理。首先判断扩展部分的类型，主要有 RREP_EXT（1）和 RREP_INET_DEST_EXT（4）两部分。如果是前者就简单写入INFO日志；如果是后者，哦对了，只有配置了网关才可能出现后者，这说明RREP的目的地址是网关的地址，而扩展部分持有真正的地址，把真的地址复制给 inet_dest_addr 变量，写日志，把 rt_flags 加上RT_GATEWAY 标识，将inet_rrep设为1。接着在循环里增加 extlen 的长度，增量为一个 AODV_ext 的大小，让 ext 指向下一个 ext 的部分。
```



### 检查是否需要创建转发路由

```
316~317: 找到到达 rrep 目的端的路由表项，交给变量 fwd_rt ，找到到达 rrep 的路由表项，交给变量 rev_rt。
321~332: 如果转发表项不存在，调用 rt_table_insert 新建一个。
323~334: 
```

