---
title: LVS  学习： Ct 深入理解
date: 2021-08-05 14:55:53
categories: 
  - [eBPF]
tags:
  - ebpf
  - lvs
author: Jony
---

# 连接跟踪表

连接跟踪也是最近学习下来的最关心、最有疑问的的一个问题了，但是应该属于一劳永逸的问题。

问题如下：
lvs 是个四层负载均衡转发器，负责将数据包转给后端的服务器。根据调度规则将数据包转发给正确的后端 Server。
因为每个 TCP 连接需要包含若干的数据包进行交互，如果每个 Packet 都按照调度规则来调度，比如按照 RR 模式来进行调度，
那么该 TCP 连接相关的若干数据包则被轮询转发给了 N 个后端服务器，那么整个数据包就乱套了。显然负载均衡需要将同属
于同一个 TCP 连接的数据流的若干数据包转发至其中一台后端服务器上，而不是分散到多个服务器。


解决方案：
lvs 若想正确转发数据包，需要维护记录数据包五元组代表的连接信息，包括首次建立连接时调度的 RS。
那么后续的同一个连接（会话）的数据包到来后，查询系统维护的连接信息，查到该连接后就能确定这条连接对应的 RS 服务器，
最终将数据包正确转发给服务器，这样就完成了将一个 TCP 连接转发给后端服务器了。

__复制博客 https://www.linuxblogs.cn/articles/20011018.html__


看这个字面意思好像根据五元组将同一个连接的请求打到同一个服务其上，从 L4 负载感觉不是很均衡啊。 和理想的有点差异的样子

lvs 标识一个连接信息，结构体如下:

```c
struct ip_vs_conn {
  struct hlist_node c_list;         /* 哈希链表头 */
  /* Protocol, addresses and port numbers */
  __be16                  cport;          /* 客户端请求源端口 */
  __be16                  dport;          /* 后端服务器的端口，可以不同于 vport */
  __be16                  vport;          /* 访问的业务端口，一般为 80 或 443 */
  u16     af;   /* 协议族，代表 v4 或 v6 */
  union nf_inet_addr      caddr;          /* 客户端 IP 地址 */
  union nf_inet_addr      vaddr;          /* 客户端访问的 vip */
  union nf_inet_addr      daddr;          /* 后端 RS 服务器 IP 地址 */
  volatile __u32          flags;          /* status flags */
  __u16                   protocol;       /* Which protocol (TCP/UDP) */
  __u16     daf;    /* dest 协议族，代表 v4 或 v6 */
  struct netns_ipvs *ipvs;

  /* counter and timer */
  refcount_t    refcnt;   /* 引用计数 */
  struct timer_list timer;    /* 连接对应的定时器 */
  volatile unsigned long  timeout;  /* 超时时间 */

  /* Flags and state transition */
  spinlock_t              lock;           /* 状态转化时需要锁操作 */
  volatile __u16          state;          /* 状态信息 */
  volatile __u16          old_state;      /* 保存上一个状态
             */
  __u32     fwmark;   /* Fire wall mark from skb */
  unsigned long   sync_endtime; /* jiffies + sent_retries */

  /* Control members */
  struct ip_vs_conn       *control;       /* Master control connection */
  atomic_t                n_control;      /* Number of controlled ones */
  struct ip_vs_dest       *dest;          /* real server */
  atomic_t                in_pkts;        /* incoming packet counter */

  /* Packet transmitter for different forwarding methods.  If it
   * mangles the packet, it must return NF_DROP or better NF_STOLEN,
   * otherwise this must be changed to a sk_buff **.
   * NF_ACCEPT can be returned when destination is local.
   */
  int (*packet_xmit)(struct sk_buff *skb, struct ip_vs_conn *cp,
         struct ip_vs_protocol *pp, struct ip_vs_iphdr *iph);

  struct ip_vs_app        *app;           /* bound ip_vs_app object */
  void                    *app_data;      /* Application private data */
  struct ip_vs_seq        in_seq;         /* incoming seq. struct */
  struct ip_vs_seq        out_seq;        /* outgoing seq. struct */

  const struct ip_vs_pe *pe;
  char      *pe_data;
  __u8      pe_data_len;

  struct rcu_head   rcu_head;
};
```
通过这些信息就完成了一个链接表管理。
**反思：好像和 conntrack 不太一样**


lvs 为了提高性能，采用了 hash 表存储，根据 `caddr`、`cport`、`proto` 关键字计算，得到一个 hash 值。

```c

ip_vs_conn_hashkey(p->ipvs, p->af, p->protocol, addr, port);

static unsigned int ip_vs_conn_hashkey(struct netns_ipvs *ipvs, int af, unsigned int proto,
               const union nf_inet_addr *addr,
               __be16 port)
```
然后再根据 hash 下面的链表计算除槽位，相当于 二次 hash 的意思。找对应的连接。

关于连接表的建立：客户端发起  `syn` 建立连接的时候触发新建连接，相同连接的话其他数据包到来时，只需要查找存在的连接表即可继续后面的转发工作。

```c

  ...
  // 只有在成功调度可用的后端 RS 服务器后才去新建连接
  cp = ip_vs_conn_new(param, type, daddr, dport, flags, dest,
            fwmark);
  ...


struct ip_vs_conn *
ip_vs_conn_new(const struct ip_vs_conn_param *p, int dest_af,
         const union nf_inet_addr *daddr, __be16 dport, unsigned int flags,
         struct ip_vs_dest *dest, __u32 fwmark)
```

连接创建步骤：
1. 分配 `连接` 所需要的内存资源空间
2. 初始化链表相关字段
3. 设置注册定时器工作函数
4. 填充核心关键字段
5. 关联绑定调度到的后端 RS 服务器
6. 初始化状态、超时时间、绑定 xmit 函数
7. 准备就绪，将 new_conn 挂再全局的哈希表上


后续的数据包就可以通过 `标识参数` 查找**连接表**。 找到改数据包所属的连接信息，也就能确定本次数据包需要转发给后端内的那一台 RS 服务器，
通过相关操作后将数据包转发给连接所对应的 目标服务器。

如果使用 `DR 模式` ，只有客户端请求的流量会经过 LVS 服务器，数据包到达 LVS 会触发下面的函数：

```c
struct ip_vs_conn_param {
  struct netns_ipvs   *ipvs;
  const union nf_inet_addr  *caddr;
  const union nf_inet_addr  *vaddr;
  __be16        cport;
  __be16        vport;
  __u16       protocol;
  u16       af;

  const struct ip_vs_pe   *pe;
  char        *pe_data;
  __u8        pe_data_len;
};

struct ip_vs_conn *ip_vs_conn_in_get(const struct ip_vs_conn_param *p)


// 查找表达式
static inline struct ip_vs_conn *
__ip_vs_conn_in_get(const struct ip_vs_conn_param *p)
{
  unsigned int hash;
  struct ip_vs_conn *cp;
  // 获取hash
  hash = ip_vs_conn_hashkey_param(p, false);

  rcu_read_lock();
  // 根据hash 查找链表桶
  hlist_for_each_entry_rcu(cp, &ip_vs_conn_tab[hash], c_list) {
    if (p->cport == cp->cport && p->vport == cp->vport &&
        cp->af == p->af &&
        ip_vs_addr_equal(p->af, p->caddr, &cp->caddr) &&
        ip_vs_addr_equal(p->af, p->vaddr, &cp->vaddr) &&
        ((!p->cport) ^ (!(cp->flags & IP_VS_CONN_F_NO_CPORT))) &&
        p->protocol == cp->protocol &&
        cp->ipvs == p->ipvs) {
      if (!__ip_vs_conn_get(cp))
        continue;
      /* HIT */
      rcu_read_unlock();
      return cp;
    }
  }

  rcu_read_unlock();

  return NULL;
}

```

1. 通过给定参数获取 hash 值，根据哈希值获取链表数据
2. 遍历 `ip_vs_conn_tab[hash]` 链表数据，查找除真实的连接


关于锁的问题：


我们现在生产环境使用的服务器都是多核处理器，在 lvs 转发处理逻辑中需要频繁的查连接表，由于连接表是全局的一张表结构，
绝大部分时间 N 个 cpu 会同时操作（包括查找、新建和删除）连接表，这样会导致连接表不一致的问题。因此，每个 cpu 
对连接表进行相关操作时需要加上相应的 **`ReadLock 或 WriteLock`**，才能保证连接表的原子性操作。

- **全局锁操作**。假如我们为连接表定义一个全局锁，每个 cpu 在对连接表做读写操作时，
都需要先加锁，获取锁后进行相关操作，操作完毕再释放锁资源。这样在高并发的业务场景下，
同一时刻锁只能被一个 cpu 占用，因此各 cpu 之间对锁的竞争会非常严重，导致整体 lvs 性能会随着 cpu 个数增加而逐渐下降。

- **局部锁优化**。lvs 在设计上没有使用全局的锁，而是根据连接表的哈希结构，为 N 个哈希桶设置一个 Lock，这里的 N 为 256，
也就是 256 个桶使用一个锁，总共有 4096 个哈希桶。如果有 16 个 cpu 的话，那么同一时间内平均每个 cpu 都可能占有一个锁，
从系统整体上看，cpu 对锁操作的竞争就会大大减弱，提高了系统整体的转发能力。


- **为每个桶绑定一个锁，进一步优化**。局部优化能够很大程度上提高系统性能，同样思路，系统资源允许的情况下，我们可以为每
个哈希桶分配设置一个 “锁”，那么各 cpu 之间锁竞争就会变得更小啦，系统性能肯定会有更好地优化。

- **无锁化，每个 cpu 一张连接表**。既然每个 cpu 都要访问连接表，且会引起锁的竞争恶化，为何不给每个 cpu 维护一张连接表，
每个 cpu 只维护自己相关数据连接组成的连接表，每个 cpu 读写 连接表时互不干扰，这样就实现无锁化，完美的避开了锁。
    - 每个 cpu 一张连接表
    - 网卡按照 4 元组哈希，同一个数据流肯定会打到同一个 cpu 上去
    - *对于 NAT 模式来说，就比较麻烦不好处理*
    主要原因： NAT 回包还会经过 lvs，回包到达 lvs 网卡时，按照 4 元哈希后，数据包可能到达网卡队列 2 了，
    而近来的请求数据包到达网卡队列 1 了，那么同一个连接的数据包不在同个 cpu 上就会出现异常现象了，
    因为回包到达的 cpu 无法处理该数据包


总结：总体来说 conntrack 状态维护最基础通过五元组来完成。将一个 TCP 内的流量打到同一台机器，不同次的请求也同样会
打到同一台，初始感觉不合理，可能和思维定位在 L7 有关系。将每次请求都转发至不同 RS 上，但是相对 L4 层，每个连接就是每次
请求，通过两个五元组决定转发方向和转发目标。也不会影响到流量。
考虑：优化如果每次报文发送都是连续性的



# 源码分析

总结下来 LVS 依然是在 netfilter 上展开的，但是相对于 LVS 来说核心概念主要涉及连接表和调度算法。它们是 LVS 的核心内容

netfilter 的 HOOK 点对应于内核中的方法：
- `NF_INET_PRE_ROUTING` 这是所有入栈数据包的第一个挂载点，挂载点位于 `ip_rcv` 中，v6 的是 `ip6_rcf`
- `NF_INET_LOCAL_IN` 对于所有发送到当前主机的入栈数据包，经过挂载点 `NF_INET_PRE_ROUTING` 并执行路由选择，挂载点位于方法 `ip_local_deliver` 中
- `NF_INET_FORWARD` 对于所有转发的数据包，经过挂载点 `NF_INET_PRE_ROUTING` 并执行路由选择子系统查找后。挂载点位于 `ip_forward` 中
- `NF_INET_POST_ROUTING` 所有要转发的数据包和当前主机生成的数据包都经过这个挂载点，挂载点位于 `ip_output` 中
- `NF_INET_LOCAL_OUT` 当前主机生成的所有出战数据包都经过这个挂载点，挂载点位于方法 `__ip_local_out` 中

```c
static const struct nf_hook_ops ip_vs_ops4[] = {
  /* 包过滤后，只更改VS/NAT的 source */
  {
    .hook   = ip_vs_reply4,
    .pf   = NFPROTO_IPV4,
    .hooknum  = NF_INET_LOCAL_IN,
    .priority = NF_IP_PRI_NAT_SRC - 2,
  },
  /* 包过滤后，通过VS/DR、VS/TUN或VS/NAT(change destination)转发包，
   * 这样过滤规则就可以应用于IPVS。 */
  {
    .hook   = ip_vs_remote_request4,
    .pf   = NFPROTO_IPV4,
    .hooknum  = NF_INET_LOCAL_IN,
    .priority = NF_IP_PRI_NAT_SRC - 1,
  },
  /* 在 ip_vs_in 之前，只为 VS/NAT 更改source */
  {
    .hook   = ip_vs_local_reply4,
    .pf   = NFPROTO_IPV4,
    .hooknum  = NF_INET_LOCAL_OUT,
    .priority = NF_IP_PRI_NAT_DST + 1,
  },
  /* 在 mangle 之后，调度和转发本地请求 */
  {
    .hook   = ip_vs_local_request4,
    .pf   = NFPROTO_IPV4,
    .hooknum  = NF_INET_LOCAL_OUT,
    .priority = NF_IP_PRI_NAT_DST + 2,
  },
  /* 在包过滤之后（但在 ip_vs_out_icmp 之前），
  *  捕获发往 0.0.0.0/0 的 icmp，用于传入的 IPVS 连接 */
  {
    .hook   = ip_vs_forward_icmp,
    .pf   = NFPROTO_IPV4,
    .hooknum  = NF_INET_FORWARD,
    .priority = 99,
  },
  /* 包过滤后，只更改VS/NAT的 source */
  {
    .hook   = ip_vs_reply4,
    .pf   = NFPROTO_IPV4,
    .hooknum  = NF_INET_FORWARD,
    .priority = 100,
  },
};
```

这里的 hook 函数是 `ip_vs_in` 是数据包的入口位置。从上边注册代码段中，可以知道它注册的挂载点位置是 `NF_INET_LOCAL_IN` 已经通过了
prerouting 和 filter 路由查找的过程。
假设在 DR 模式下，数据包转发将会经过如下过程：
1. `[pp = ip_vs_proto_get(iph.protocol);](https://elixir.bootlin.com/linux/v5.11.2/source/net/netfilter/xt_ipvs.c#L81)` 查找是否是 LVS 所支持的协议
   lvs 所支持。lvs 能够支持 tcp、udp、ah、esp 协议，如果不是 LVS  所支持的协议，那么将直接返回 NF_ACCEPT 也就没有继续执行下去的意义了
2. 调度并新建连接信息 
  根据数据包 5 元组相关字段进行查找 `conn_in_get`，这里假设是一个新连接的请求数据包，那么在连接表中是查不到连接表项，将会调用 `conn_schedule` 新建表项
  新建表现的条件：
  - 数据包必须携带 SYN 标记
  - 数据包所访问请求必须是我们定义的 service 才行
  满足上述条件之后，会调用 `ip_vs_schedule` 方法，从 svc 指向的 dests 中更具调度算法选出一个合适的后端服务器
3. 在成功调度 dest 后，调用 `ip_vs_conn_new` 函数来常见真正的连接条目：
  - 分配 cp 内存资源
  - 设置定时器
  - 填充相关字段
  - 绑定并设置 dest 相关信息，调用 `ip_vs_bind_dest` 函数
  - 绑定 conn 所需的 xmit 函数，调用 `ip_vs_bind_xmit` 函数
  - 将 cp 连接条目 hash 到全局的连接表中


```c
net/netfilter/ipvs/ip_vs_app.c
net/netfilter/ipvs/ip_vs_conn.c
net/netfilter/ipvs/ip_vs_core.c
net/netfilter/ipvs/ip_vs_ctl.c
net/netfilter/ipvs/ip_vs_dh.c         目的地址哈希调度
net/netfilter/ipvs/ip_vs_est.c        简单的速率估计
net/netfilter/ipvs/ip_vs_fo.c         添加了基于权重的初始功能
net/netfilter/ipvs/ip_vs_ftp.c        ftp应用模块
net/netfilter/ipvs/ip_vs_lblc.c           基于局部性的最少连接
net/netfilter/ipvs/ip_vs_lblcr.c      带复制的基于局部性最少链接
net/netfilter/ipvs/ip_vs_lc.c         最少连接调度
net/netfilter/ipvs/ip_vs_mh.c         磁悬浮哈希调度模块
net/netfilter/ipvs/ip_vs_nfct.c       支持IPVS的Netfilter连接跟踪
net/netfilter/ipvs/ip_vs_nq.c         永不排队调度算法
net/netfilter/ipvs/ip_vs_ovf.c        溢出连接调度模块
net/netfilter/ipvs/ip_vs_pe.c         持久化模块
net/netfilter/ipvs/ip_vs_pe_sip.c
net/netfilter/ipvs/ip_vs_proto.c      transport协议支持ipv4负载均衡
net/netfilter/ipvs/ip_vs_proto_ah_esp.c   针对IPV的AH/ESP IPSec负载平衡支持
net/netfilter/ipvs/ip_vs_proto_tcp.c      IPVS的TCP负载平衡支持
net/netfilter/ipvs/ip_vs_proto_udp.c    IPVS的UDP负载平衡支持
net/netfilter/ipvs/ip_vs_rr.c           轮询调度算法
net/netfilter/ipvs/ip_vs_sched.c
net/netfilter/ipvs/ip_vs_sed.c          最少期望延迟调度算法
net/netfilter/ipvs/ip_vs_sh.c           源地址哈希调度
net/netfilter/ipvs/ip_vs_sync.c       
net/netfilter/ipvs/ip_vs_wlc.c          加权最少连接
net/netfilter/ipvs/ip_vs_wrr.c          加权轮询调度
net/netfilter/ipvs/ip_vs_xmit.c
```
[KMSG_COMPONENT](https://elixir.bootlin.com/linux/v5.11.2/C/ident/KMSG_COMPONENT)






