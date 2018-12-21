```
void NS_CLASS neighbor_add(AODV_msg * aodv_msg, struct in_addr source,
			   unsigned int ifindex)
{
    struct timeval now;
    rt_table_t *rt = NULL;
    u_int32_t seqno = 0;

    gettimeofday(&now, NULL);

    rt = rt_table_find(source);

    if (!rt) {
	DEBUG(LOG_DEBUG, 0, "%s new NEIGHBOR!", ip_to_str(source));
	rt = rt_table_insert(source, source, 1, 0,
			     ACTIVE_ROUTE_TIMEOUT, VALID, 0, ifindex);
    } else {
	/* Don't update anything if this is a uni-directional link... */
	if (rt->flags & RT_UNIDIR)
	    return;

	if (rt->dest_seqno != 0)
	    seqno = rt->dest_seqno;

	rt_table_update(rt, source, 1, seqno, ACTIVE_ROUTE_TIMEOUT,
			VALID, rt->flags);
    }

    if (!llfeedback && rt->hello_timer.used)
	hello_update_timeout(rt, &now, ALLOWED_HELLO_LOSS * HELLO_INTERVAL);

    return;
}
```

12-15 查找节点到达该邻居的路由链路，若不存在则新建链路并加入路由表

18-19若存在路由链路且为单向则不做更新（flags为16位，与RT_UNIDIR做位与操作，若结果为1则为单项）

21-26若存在路由链路且为双向则更新该路由链路（包括目的节点序列号等）

28-29若正在发送hello报文（存在报文计时器），则更新timeout时间（因rt为active）





precursor list是针对每一条路由表项而言，当当前节点需要转发或产生reply时，记下使用了这条路由链路的邻居节点（放在precursor list里）。当该链路发生lose事件时，此precursor list中的邻居节点即收到RERR报文信息。

	if (rt->nprec == 1)
	    rerr_unicast_dest = FIRST_PREC(rt->precursors)->neighbor;
	}
    /* Purge precursor list: */
    if (!(rt->flags & RT_REPAIR))
    precursor_list_destroy(rt);
问题：

rt->precursors在FIRST定义下指向前驱节点的list_t,后被强制转化为precursor list。但强制转化后neighbor为？

precursor list 中什么时候加入的前驱节点

还没发送rerr即将precursor list删除？后广播？











