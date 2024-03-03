![img](https://mmbiz.qpic.cn/mmbiz_png/E8yb1gQ3rpAic3tBUlFCmHJcZCibiaWib893fhYHQwwVICNHtazn3hDB1JiaUcb2MibdqTDznyGw9PVGGCeDWv00YvNA/640?wx_fmt=png)

最近在学习K8S，一如既往，先从网络入手以输出倒逼输入。开启K8S网络系列的第一篇：



**K8S几种网络解决方案大都依赖iptables，所以先介绍iptables基本概念和操作：**
​

1、iptables的几张表和规则链



2、手写一条rules展示nat实现：

（在POSTROUTING插入一条规则，将源地址为192.168.50.0/24的数据包做SNAT动作，将源地址替换为192.168.1.11）

```
# iptables -t nat -A POSTROUTING -s 192.168.50.0/24 -j SNAT --to-source 192.168.1.11
```

查看刚添加的规则

```
# iptables -t nat -nvL Chain POSTROUTING (policy ACCEPT 1 packets, 60 bytes) pkts bytes target     prot opt in     out     source               destination 0     0     SNAT      all  --  *      *       192.168.50.0/24      0.0.0.0/0            to:192.168.1.11
```



3、规则链之间可以“跳转”



4、查看iptables单个规则链

```
# iptables -t nat -nvL POSTROUTINGChain POSTROUTING (policy ACCEPT 1 packets, 60 bytes) pkts bytes target     prot opt in     out     source               destination 0     0     SNAT      all  --  *      *       192.168.50.0/24      0.0.0.0/0            to:192.168.1.11
```







**K8S的网络模型（待定，未掌握，可删）**





**K8S的NAT网络 iptables实例**

**
**

**
**

**K8S原生负载均衡 SVC**







**SVC iptables负载均衡过程分解：**

（以client访问K8S内nginx服务集群为例，由外向内逐条分析iptables rules）

1、查看table nat 的 PREROUTING链，可见K8S生成一条默认规则，将0.0.0.0/0->0.0.0.0/0的all protocol指向了KUBE-SERVICES链，依次实现了K8S接管所有由外向内的nat访问

```
# iptables -t nat  -nvL PREROUTINGChain PREROUTING (policy ACCEPT 23 packets, 5876 bytes) pkts bytes target                     prot opt in     out     source               destination  25  5996 KUBE-SERVICES               all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals *
```

2、查看nat 表的KUBE-SERVICES链，可见三条rules：

```
Chain KUBE-SERVICES (2 references) pkts bytes target                        prot opt in     out     source               destination    0     0 KUBE-MARK-MASQ                tcp  --  *      *      !10.244.0.0/16        10.96.129.200        /* default/my-ngx-deploy-svc:80-80 cluster IP */ tcp dpt:80    0     0 KUBE-SVC-LNJBOIUQQNHIUUAM     tcp  --  *      *       0.0.0.0/0            10.96.129.200        /* default/my-ngx-deploy-svc:80-80 cluster IP */ tcp dpt:80    0     0 KUBE-NODEPORTS                all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service nodeports;  */ ADDRTYPE match dst-type LOCAL                                                                                                               NOTE: this must be the last rule in this chain
```

  rule1：将源地址**非**10.244.0.0/16，目的地址为负载均衡器地址(10.96.129.200)的流量重新标记

  rule2：将所有访问负载均衡器地址(10.96.129.200)的流量指向KUBE-SVC-LNJBOIUQQNHIUUAM链，该链由负载均衡器SVC生成

  rule3:将其它流量指向KUBE-NODEPORTS链

先忽略rule1、3，重点关注指向SVC链的rule2



3、查看nat表的KUBE-SVC-LNJBOIUQQNHIUUAM链：

```
Chain KUBE-SVC-LNJBOIUQQNHIUUAM (2 references) pkts bytes target                        prot opt in     out     source               destination    0     0 KUBE-SEP-4NSVBZT5YEFFSKVT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/my-ngx-deploy-svc:80-80 */ statistic mode random probability 0.20000000019    0     0 KUBE-SEP-VBHCWHVRVI4ZJ6BF     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/my-ngx-deploy-svc:80-80 */ statistic mode random probability 0.25000000000    0     0 KUBE-SEP-AWVZXWRX3KOQFAVV     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/my-ngx-deploy-svc:80-80 */ statistic mode random probability 0.33333333349    0     0 KUBE-SEP-CKLXYKSZB7DONXIR     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/my-ngx-deploy-svc:80-80 */ statistic mode random probability 0.50000000000    0     0 KUBE-SEP-A4Q24IDGRSRKP54I     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/my-ngx-deploy-svc:80-80 */
```

我创建了5个nginx POD，因此可见5条负载均衡的rules。

当流量依次经过 PREROUTING ->KUBE-SERVICES ->KUBE-SVC-LNJBOIUQQNHIUUAM后将会被5条rules负载均衡到5个chain(KUBE-SEP-xxx) ，



4、以前两个KUBE-SEP-xxx为例，查看最终动作：

```
Chain KUBE-SEP-4NSVBZT5YEFFSKVT (1 references) pkts bytes target                        prot opt in     out     source               destination    0     0 KUBE-MARK-MASQ                all  --  *      *       10.244.0.63          0.0.0.0/0            /* default/my-ngx-deploy-svc:80-80 */    0     0 DNAT                          tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/my-ngx-deploy-svc:80-80 */ tcp to:10.244.0.63:80Chain KUBE-SEP-VBHCWHVRVI4ZJ6BF (1 references) pkts bytes target                        prot opt in     out     source               destination    0     0 KUBE-MARK-MASQ                all  --  *      *       10.244.0.65          0.0.0.0/0            /* default/my-ngx-deploy-svc:80-80 */    0     0 DNAT                          tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/my-ngx-deploy-svc:80-80 */ tcp to:10.244.0.65:80
```

可以看到每个chain包含两条rule，对于外部流量来说起作用的是rule2

rule2将所有TCP流量执行DNAT动作，将目的地址和端口转换为10.244.0.63:80和10.244.0.65:80

至此，iptables实现的负载均衡完成。



-> -> -> -> ->