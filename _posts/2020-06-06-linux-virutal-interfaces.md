---
title: 'Linux虚拟网络接口'
date: 2020-06-06
author: csulrong
version: 3.4.3
categories: [linux]
tags: [networking]
excerpt_separator: <!-- more -->
---

Linux有丰富的虚拟化网络功能，当你在Linux系统中执行`ip link help`命令的时候，就会发现Linux支持很多种类型的虚拟网络接口，这些虚拟网络接口把虚拟机、容器以及裸金属宿主机互联起来，为云计算网络虚拟环境提供了技术支撑。本文重点介绍Linux中各种常用的虚拟网络接口，为在网络虚拟化领域学习和工作的同行们提供参考。

<!--more-->

本文重点介绍以下几种常用的虚拟网络接口：
- 虚拟网桥：bridge
- 接口捆绑：bonded和team
- VLAN：vlan、macvlan、ipvlan
- 虚拟路由和转发：vrf
- Overlay与Tunnel：gre、vxlan、geneve
- macvtap、ipvtap
- veth
- dummy
- nlmon
- netdevsim


## 虚拟网桥 (bridge)

Linux bridge类似于一台网络交换机，把一台机器上的若干个物理或者虚拟的网络接口“连接”起来，负责相连网络接口之间的数据包的二层转发，既可用于和外部路由器或者网关之间收发数据包，也用于转发位于同一台主机上不同虚拟机(VMs)或网络命名空间(network namespace)之间的数据包。Linux bridge支持STP, VLAN filter以及组播侦听(multicast snooping)。

{: .align-center}
![Linux虚拟网桥]({{ site.baseurl }}/assets/images/2020/linux_bridge.svg)

使用下面的`ip link`命令，创建如上图所示的虚拟网桥`br0`，并将主机的物理接口`eth0`、连接虚拟机的`tap1`和`tap2`、以及网络命名空间的`veth1`虚拟接口连接到`br0`上。

{% highlight bash %}
ip link add br0 type bridge
ip link set eth0 master br0
ip link set tap1 master br0
ip link set tap2 master br0
ip link set veth1 master br0
{% endhighlight %}

<div class="note info">
  <p>也可以使用`bridge-utils`包中`brctl`命令创建和管理虚拟网桥。</p>
</div>

## 接口捆绑：bonded和team

接口捆绑将多个网络接口组合成一个逻辑接口，从而提供冗余和带宽聚合。Linux提供了bonded和team两个driver。

{: .align-center}
![接口捆绑]({{ site.baseurl }}/assets/images/2020/interface_bundle.svg)

bonding和teaming这两个术语因为都跟接口捆绑或链路聚合有关，它们也经常被混为一谈，但Linux在实现上是有区别的。

使用如下命令，创建一个bonded接口`bond1`，将主机上的两个物理接口`eth0`和`eth1`捆绑成一个逻辑的聚合接口，该聚合接口工作在主备模式，提供了链路的冗余和备份。

{% highlight bash %}
ip link add bond1 type bond miimon 100 mode active-backup
ip link set eth0 master bond1
ip link set eth1 master bond1
{% endhighlight %}

teaming可以作为接口聚合的首选方案，因为它提供了比bonding更全面的功能和特性。关于两者在功能特性上的具体区别，可以参考[Comparison of network teaming and bonding features](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/configuring-network-teaming_configuring-and-managing-networking#comparison-of-network-teaming-and-bonding-features_configuring-network-teaming)

在下面的命令行中，使用`teamd`创建一个`team0`聚合接口，并将主机上的`eth0`和`eth1`加入到这个聚合接口。

{% highlight bash %}
teamd -o -n -U -d -t team0 -c '{"runner": {"name": "loadbalance"},"link_watch": {"name": "ethtool"}}'
ip link set eth0 down
ip link set eth1 down
teamdctl team0 port add eth0
teamdctl team0 port add eth1
{% endhighlight %}

## VLAN技术：vlan、macvlan、ipvlan

VLAN通过在数据帧上加标签的方式，将一个物理局域网在逻辑上划分成多个广播域，实现广播域的隔离。同一个VLAN内的主机间可以直接通信，而不同VLAN之间不能直接通信，从而将广播报文限制在一个VLAN内。

{: .align-center}
![数据帧的VLAN封装格式]({{ site.baseurl }}/assets/images/2020/vlan.svg)


## 参考文献

[Introduction to Linux interfaces for virtual networking](https://developers.redhat.com/blog/2018/10/22/introduction-to-linux-interfaces-for-virtual-networking/)


-----------------
