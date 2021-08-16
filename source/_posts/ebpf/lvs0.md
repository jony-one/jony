---
title: LVS  学习： 操作实践
date: 2021-08-10 11:18:19
categories: 
	- [eBPF]
tags:
  - ebpf
  - lvs
author: Jony
---



LVS 常用的分为三个模式：NAT、Tunnel、DR。

在参考 [负载均衡 LVS 入门教程详解 - 操作实践](https://www.linuxblogs.cn/articles/20010720.html) 进行 DR 模式实操时，文档缺少了对 VIP 的绑定导致 VIP 对
外一直无法访问，最后添加了对 VIP 的绑定 eth 可以访问通过。

## DR 模式

DR 模式通过只修改目标 MAC 地址来完成请求的转发，响应

问题一：为什么 DR 模式下，VIP 绑定都绑定在 lo 上，而不是 eth0 等其他网卡上，同样可以使得 ARP 屏蔽外网探测？

答：在 lo 上配置 VIP 后，正常情况下，其他设备发送 VIP 的 arp 请求时，除了负载均衡设备会响应 ARP 请之外，配置了 VIP 的 RealServer 也会响应 VIP  的 arp 请求这样请求设备
就无法判断那个设备时真实负载了。
解决 lo 挂载了 VIP ，不需要对外提供 arp 响应的问题，需要使用配置 `arp_ignore=1` ，这样后端服务就只会响应目的 IP 地址为接收网卡上的本地地址的 arp 请求。
**紧接着的问题**：知道配置 `arp_ignore` 的作用，但是不知道为什么这么配置

arp_ignore：主要用于 arp 请求的响应策略，策略值范围在 0\-8 之间，3\~8使用较少
- 0 - (默认值): 回应任何网络接口（网卡）上对任何本机IP地址的arp查询请求。
- 1：只响应目的IP地址为接收网卡上的本地地址的arp请求。
- 2：只响应目的IP地址为接收网卡上的本地地址的arp请求，并且**arp请求的源IP必须和接收网卡同`网段`**。
- 3：如果ARP请求数据包所请求的IP地址对应的本地地址其作用域（scope）为主机（host），则不回应ARP响应数据包，如果作用域为全局（global）或链路（link），则回应ARP响应数据包。
- 4\~7：保留未使用
- 8：不回应所有的arp请求

arp_announce：主要用用于`对外 arp 请求策略`，发送 arp 请求时使用选择源 IP 地址策略。取值范围：0\-3
- 0：允许使用任意网卡上的IP地址作为arp请求的源IP，通常就是使用数据包a的源IP。
- 1：尽量避免使用 `不属于该发送网卡子网` 的本地地址作为发送arp请求的源IP地址。
- 2：忽略IP数据包的源IP地址，选择该发送网卡上最合适的本地地址作为 `arp` 请求的 `源IP` 地址。

上述配置在路径:`/proc/sys/net/ipv4/conf/` 下面，通常除了可见的网卡配置以外还有 `default` 和 `all` 两个网卡属性，如果网卡没有配置 
`arp_ignore` 和 `arp_announce` 的情况下会自动继承 `default` 的值。



问题二：在使用 LVS 时是如何保存状态的，每个连接发送的报文都是独立，如何做到一个请求下的报文发送到同一个 RealServer？
答：探索中.....



## NAT 模式

除了完成正常的 LVS 配置以外还需要配置 RS 方的网关。

关于 NAT 模式，理解将 LVS 理解为一个 7 层网关就容易理解其工作模式，通过修改器目标 IP 完成请求、响应

问题三：通过文章解释 NAT 是因为 RS 上默认网关配置为 LVS 设备 IP ，所以才会将响应包回流到 LVS 上，为什么不是修改 CIP 做 CIP 映射？
答：后端服务器需要将响应报文发送给 LVS 设备，此时的数据包目的 IP 是客户端的 IP，可能是外网的任何一个网段，需要通过设置路由的方式，将数据包转发给 LVS ，因为不可能穷举所有的网段地址，所以必须要设置默认的网关来指向 LVS。所以也就解释了为啥要设置网关。

问：但是为什么不能修改 CIP
答：探索中...

落地有点：配置简单，不需要什么风险配置，可跨机房。
缺点：存在性能瓶颈


## tunnel 模式


tunnel 于 dr 模式相当都是单项负载均衡，性能也很强。支持跨机房通信

操作模式：当确定 IPVS 服务之后，会选择一个 RS 作为后端服务器，以 RIP 为目的 IP 查找路由信息，确定下一条、DEV 等信息，然后再 IP 头部前面额外增加一个 IP 头：
以 DIP 为源，RIP 为目的 IP，将数据发送到 output 行。
数据包经过 LVS 网关发送至路由，转发至后端服务。

问题四：再对加入额外 IP 头的时候，不久增加报文长度了，是不是需要重新排序、切割报文信息然后进行转发
答：这也是 Tunnel 的一个缺点，额外的头部加入之后会导致新的分片，影响服务器性能。而且后端需要安装 ipip 模块还需要配置 tunnel ，进行额外 IP 头剥离操作。tunnel 也不支持端口映射。

问题五：关于分别什么时候使用 DR、NAT、Tunnel 模式？
答：
1. 对转发性能要求较高，且有跨机房需求，Tunnel 可能是较好的选择。
2. 如果存在 windows 系统，使用 lvs 的话，则必须选择 NAT 模式了。
3. 如果对性能要求非常高，可以首选 DR 模式，而且可以透传客户端源 IP 地址。


问题六：如果不考虑性能的话是不是 NAT 更通用、简洁，那么如何优化 NAT 性能。
答：经过文档查阅发现，并不会优雅，NAT 模式只是 DNA 模式，对 Source IP  并不是很友好，相对零配置来说还是存在比较高的维护成本。
为了将配置和维护成本降至最低可以采用 FullNAT 模式，在 LVS 中使用 SNAT  将来源 IP 改为内网IP ，使用 DNAT 将目标IP 修改进来，
但是存在了大约 NAT 的 10% 性能损耗，也就是说零成本高性能的模式目前这种模式处于理想状态，还没有实现
[FullNAT](https://github.com/alibaba/LVS/blob/master/docs/LVS_FULLNAT.pdf)


## 关于 keepalived

使用 ipvsadm 配置有几个缺点：

1. RS 如果宕机或者无响应，LVS 仍然会将请求转发到后端，导致异常流量
2. 单点问题，如果单点 LVS 服务器宕机，则全部服务不可用

Keepalived是基于vrrp协议的一款高可用软件。Keepailived有一台主服务器和多台备份服务器，在主服务器和备份服务器上面部署相同的服务配置，使用一个虚拟IP地址对外提供服务，
当主服务器出现故障时，虚拟IP地址会自动漂移到备份服务器。容灾策略



问题七：除了 keepalived 通用性，是否存在 LVS 控制平面？
暂时没有发现，keepalived 是 LVS 的集群部署平面，当集群在服务时会通过心跳机制保持 LVS 的在线状态，
如果其中一个 LVS 宕机，keepalived 将会感知到，并且 VIP 将会被漂移至健康的实例上


问题八：ipvsadm 和 iptables 都是操作 netfilter ，那么他们两个有什么区别？
iptables 是基于一条条的规则列表。而且初始目的是为了 **防火墙** 设计的，集群数量越多规则就越多，匹配规则的时间是 O(n) 占用大量的 CPU 时间。
问：iptables 是否是规则密集集中型？就是80% 的规则集中再 20% 的负载均衡上
iptables 是规则匹配模式，数据包进入需要经过 raw-filter-input 都进需要遍历很多规则，很多规则都是重复性的。包括 iptables 的 NAT 规则都是
在对规则进行匹配，修改只是一瞬间的事情。所以整个 iptables 的规则判断 95% 都是在循环规则。


IPVS 是基于 HASH 表的，获取规则的时间总是 `O(1)` 与集群规模无关。
总结：以 Kubernetes 举例，当 POD 超过十万个，规则超过一万条的情况下，iptables 开始凸显出性能劣势，被 ipvs 完全打败，如果规则在
以几何的形式增长，那么iptables 性能更差。
iptables 增长的主要原因还是规则爆炸。

iptables 弥补方案：Calico 使用的是一个很短的优化过的规则链，使用 ipsets 加持，也具备了 O(1) 的复杂查询能力，与 ipvs 相差无几。
参考：[kube-proxy 模式对比：iptables 还是 IPVS？](https://blog.fleeto.us/post/iptables-or-ipvs/)

问题九：查看代码实现，了解详细实现


问题十：ebpf 能否为其做加速？
答：可以为其加入，复用的 netfilter 网关，所以在 PREROUTING - POSTOUTING  可以直接打通，中间的所有环节都会被跳过，不在经过多次轮询
另外 iptables 虽然是遍历规则，但是可以通过 ipset 加速，从 O(n) 的查找时间减少至 O(1) 。CPU 损耗进一步降低，但是相对来说内存可能损耗较大

参考文档：[深入理解iptables]https://cctrip.tech/deep_iptables/


问题十一：如果 DR 模式不只是修改 DMAC 连着 DIP 一起修改会不会就可以直接返回数据了
答：可以这么修改，但是深度思考了一下，如果 DIP 被修改了，那么数据包回到 client 就回变成 client  直连 dip 的意思，可能会导致网络不通
或者是网络找不到主机，还是存在一定问题，除非一种解决方案就是通过本地路由器或者其他主机将 DIP 修改为 VIP，那么这个模式又像 NAT 模式了，
所以猜想不成立


问题十二：每种模式的都会修改数据包，每个数据包的被修改的 HOOK 点分别是哪里。（面试题问到）

答： [负载均衡 LVS 入门教程详解 - 基础原理](https://www.linuxblogs.cn/articles/20010419.html) 三张图就可以看出来
DR 模式流量经过  client -> PREROUTING->INPUT->OUTPUT->POSTROUTING（修改 MAC 地址）。DR 模式修改的是 MAC 地址，所以在 POSTROUTING 修改
NAT 模式流量经过 client -> PREROUTING->INPUT->（修改 IP ）OUTPUT->POSTROUTING. NAT 模式通过修改目标IP ，所以在 INPUT->OUTPUT 过程中修改
TUN 模式流量经过 client -> PREROUTING->INPUT->（添加 IP头 ）OUTPUT->POSTROUTING TUN 模式通过修改目标 IP 和 目标 MAC，所以应该涉及两处修改

总结：通过图上显示就可以看出 PREROUTING、FORWARD、POSTROUTING 都工作在链路层，INPUT、OUTPUT 工作在网络层，所以修改 MAC 地址应该在 POSTROUTING 下，
修改 IP 应该在 INPUT、OUTPUT 下。

