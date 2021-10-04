---
type: docs
title: "IP 协议"
linkTitle: "IP 协议"
weight: 2
---

## IP 及配套协议详解

因为网络层是整个互联网的核心，因此应当让网络层尽可能简单。网络层向上只提供简单灵活的、无连接的、尽最大努力交互的数据报服务。

使用 IP 协议，可以把异构的物理网络连接起来，使得在网络层看起来好像是一个统一的网络。

与 IP 协议配套使用的还有三个协议：

- 地址解析协议 ARP（Address Resolution Protocol）
- 网际控制报文协议 ICMP（Internet Control Message Protocol）
- 网际组管理协议 IGMP（Internet Group Management Protocol）

### IP 数据报格式

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502145819.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

- **版本** : 有 4（IPv4）和 6（IPv6）两个值；

- **首部长度** : 占 4 位，因此最大值为 15。值为 1 表示的是 1 个 32 位字的长度，也就是 4 字节。因为固定部分长度为 20 字节，因此该值最小为 5。如果可选字段的长度不是 4 字节的整数倍，就用尾部的填充部分来填充。

- **区分服务** : 用来获得更好的服务，一般情况下不使用。

- **总长度** : 包括首部长度和数据部分长度。

- **生存时间** ：TTL，它的存在是为了防止无法交付的数据报在互联网中不断兜圈子。以路由器跳数为单位，当 TTL 为 0 时就丢弃数据报。

- **协议** ：指出携带的数据应该上交给哪个协议进行处理，例如 ICMP、TCP、UDP 等。

- **首部检验和** ：因为数据报每经过一个路由器，都要重新计算检验和，因此检验和不包含数据部分可以减少计算的工作量。

- **标识** : 在数据报长度过长从而发生分片的情况下，相同数据报的不同分片具有相同的标识符。

- **片偏移** : 和标识符一起，用于发生分片的情况。片偏移的单位为 8 字节。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502145903.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

### IP 地址编址方式

IP 地址的编址方式经历了三个历史阶段：

- 分类
- 子网划分
- 无分类

#### 1. 分类

由两部分组成，网络号和主机号，其中不同分类具有不同的网络号长度，并且是固定的。

```
IP 地址 ::= {< 网络号 >, < 主机号 >}
```

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502145926.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

#### 2. 子网划分

通过在主机号字段中拿一部分作为子网号，把两级 IP 地址划分为三级 IP 地址。

```
IP 地址 ::= {< 网络号 >, < 子网号 >, < 主机号 >}
```

要使用子网，必须配置子网掩码。一个 B 类地址的默认子网掩码为 255.255.0.0，如果 B 类地址的子网占两个比特，那么子网掩码为 11111111 11111111 11000000 00000000，也就是 255.255.192.0。

注意，外部网络看不到子网的存在。

#### 3. 无分类

`无分类编址 CIDR` 消除了传统 A 类、B 类和 C 类地址以及划分子网的概念，使用网络前缀和主机号来对 IP 地址进行编码，网络前缀的长度可以根据需要变化。

```
IP 地址 ::= {< 网络前缀号 >, < 主机号 >}
```

CIDR 的记法上采用在 IP 地址后面加上网络前缀长度的方法，例如 128.14.35.7/20 表示前 20 位为网络前缀。

CIDR 的地址掩码可以继续称为子网掩码，子网掩码首 1 长度为网络前缀的长度。

一个 CIDR 地址块中有很多地址，一个 CIDR 表示的网络就可以表示原来的很多个网络，并且在路由表中只需要一个路由就可以代替原来的多个路由，减少了路由表项的数量。把这种通过使用网络前缀来减少路由表项的方式称为路由聚合，也称为 **构成超网** 。

在路由表中的项目由“网络前缀”和“下一跳地址”组成，在查找时可能会得到不止一个匹配结果，应当采用最长前缀匹配来确定应该匹配哪一个。

### 地址解析协议 ARP

网络层实现主机之间的通信，而链路层实现具体每段链路之间的通信。因此在通信过程中，IP 数据报的源地址和目的地址始终不变，而 MAC 地址随着链路的改变而改变。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502150019.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

ARP 实现由 IP 地址得到 MAC 地址。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502150031.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

每个主机都有一个 ARP 高速缓存，里面有本局域网上的各主机和路由器的 IP 地址到 MAC 地址的映射表。

如果主机 A 知道主机 B 的 IP 地址，但是 ARP 高速缓存中没有该 IP 地址到 MAC 地址的映射，此时主机 A 通过广播的方式发送 ARP 请求分组，主机 B 收到该请求后会发送 ARP 响应分组给主机 A 告知其 MAC 地址，随后主机 A 向其高速缓存中写入主机 B 的 IP 地址到 MAC 地址的映射。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502150045.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

### 网际控制报文协议 ICMP

ICMP 是为了更有效地转发 IP 数据报和提高交付成功的机会。它封装在 IP 数据报中，但是不属于高层协议。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502150106.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

ICMP 报文分为差错报告报文和询问报文。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502150118.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

#### 1. Ping

Ping 是 ICMP 的一个重要应用，主要用来测试两台主机之间的连通性。

Ping 的原理是通过向目的主机发送 ICMP Echo 请求报文，目的主机收到之后会发送 Echo 回答报文。Ping 会根据时间和成功响应的次数估算出数据包往返时间以及丢包率。

#### 2. Traceroute

Traceroute 是 ICMP 的另一个应用，用来跟踪一个分组从源点到终点的路径。

Traceroute 发送的 IP 数据报封装的是无法交付的 UDP 用户数据报，并由目的主机发送终点不可达差错报告报文。

源主机向目的主机发送一连串的 IP 数据报。第一个数据报 P1 的生存时间 TTL 设置为 1，当 P1 到达路径上的第一个路由器 R1 时，R1 收下它并把 TTL 减 1，此时 TTL 等于 0，R1 就把 P1 丢弃，并向源主机发送一个 ICMP 时间超过差错报告报文； 源主机接着发送第二个数据报 P2，并把 TTL 设置为 2。P2 先到达 R1，R1 收下后把 TTL 减 1 再转发给 R2，R2 收下后也把 TTL 减 1，由于此时 TTL 等于 0，R2 就丢弃 P2，并向源主机发送一个 ICMP 时间超过差错报文。

不断执行这样的步骤，直到最后一个数据报刚刚到达目的主机，主机不转发数据报，也不把 TTL 值减 1。但是因为数据报封装的是无法交付的 UDP，因此目的主机要向源主机发送 ICMP 终点不可达差错报告报文。

之后源主机知道了到达目的主机所经过的路由器 IP 地址以及到达每个路由器的往返时间。

## IPV6

我国在2014-2015年也逐步停止了向新用户和应用分配 IPv4 地址。 解决 IP 地址耗尽的根本措施就是采用具有更大地址空间的新版本的 IP，即 IPv6。 所引进的主要变化如下：

- **更大的地址空间。IPv6 将地址从 IPv4 的 32 位 增大到了 128 位**。
- 扩展的地址层次结构。
- 灵活的首部格式。 IPv6 定义了许多可选的扩展首部。
- 改进的选项。 IPv6 允许数据报包含有选项的控制信息，其选项放在有效载荷中。
- 允许协议继续扩充。
- 支持即插即用（即自动配置）。因此 IPv6 不需要使用 DHCP。
- 支持资源的预分配。  IPv6 支持实时视像等要求，保证一定的带宽和时延的应用。
- IPv6 首部改为 8 字节对齐。首部长度必须是 8 字节的整数倍。原来的 IPv4 首部是 4 字节对齐。

### 数据包格式

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502150208.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

### V4 -> V6

向 IPv6 过渡只能采用逐步演进的办法，同时，还必须使新安装的 IPv6 系统能够向后兼容：IPv6 系统必须能够接收和转发 IPv4 分组，并且能够为 IPv4 分组选择路由。

两种向 IPv6 过渡的策略：

- 使用双协议栈
- 使用隧道技术

双协议栈主机在和 IPv6 主机通信时是采用 IPv6 地址，而和 IPv4 主机通信时就采用 IPv4 地址。 根据 DNS 返回的地址类型可以确定使用 IPv4 地址还是 IPv6 地址。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502150228.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

在 IPv6 数据报要进入IPv4网络时，把 IPv6 数据报封装成为 IPv4 数据报，整个的 IPv6 数据报变成了 IPv4 数据报的数据部分。 当 IPv4 数据报离开 IPv4 网络中的隧道时，再把数据部分（即原来的 IPv6 数据报）交给主机的 IPv6 协议栈。

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502150241.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

## IP与Mac之间关系

- **整体与局部**

信息传递时候，需要知道的其实是两个地址：终点地址（Final destination address）下一跳的地址（Next hop address）IP地址本质上是终点地址，它在跳过路由器（hop）的时候不会改变，而MAC地址则是下一跳的地址，每跳过一次路由器都会改变。这就是为什么还要用MAC地址的原因之一，它起到了记录下一跳的信息的作用。注：一般来说IP地址经过路由器是不变的，不过NAT（Network address translation）例外，这也是有些人反对NAT而支持IPV6的原因之一。

- **分层实现**

如果在IP包头（header）中增加了”下一跳IP地址“这个字段，在逻辑上来说，如果IP地址够用，交换机也支持根据IP地址转发（现在的二层交换机不支持这样做），其实MAC地址并不是必要的。但用MAC地址和IP地址两个地址，用于分别表示物理地址和逻辑地址是有好处的。这样分层可以使网络层与链路层的协议更灵活地替换，网络层不一定非要用『IP』协议，链路层也不一定非用『以太网』协议。这就像OSI七层模型，TCP/IP五层模型其实也不是必要的，用双层模型甚至单层模型实现网络也不是不可以的，只是那样做很蛋疼罢了。

- **早期的『以太网』实现**

早期的以太网只有集线器（hub），没有交换机（switch），所以发出去的包能被以太网内的所有机器监听到，因此要附带上MAC地址，每个机器只需要接受与自己MAC地址相匹配的包。

## 网络地址转换NAT

> 问题：在专用网上使用专用地址的主机如何与互联网上的主机通信（并不需要加密）？

采用网络地址转换 NAT。这是目前使用得最多的方法。

装有 NAT 软件的路由器叫作 NAT路由器，它至少有一个有效的外部全球IP地址，所有使用本地地址的主机在和外界通信时，都要在 NAT 路由器上将其本地地址转换成全球 IP 地址。

通过 NAT 路由器的通信必须由专用网内的主机发起。专用网内部的主机不能充当服务器用，因为互联网上的客户无法请求专用网内的服务器提供服务。

### 转换过程

- 内部主机 A 用本地地址 IPA 和互联网上主机 B 通信所发送的数据报必须经过 NAT 路由器。
- NAT 路由器将数据报的源地址 IPA 转换成全球地址 IPG，并把转换结果记录到NAT地址转换表中，目的地址 IPB 保持不变，然后发送到互联网。
- NAT 路由器收到主机 B 发回的数据报时，知道数据报中的源地址是 IPB 而目的地址是 IPG。
- 根据 NAT 转换表，NAT 路由器将目的地址** IPG 转换为 IPA，转发给最终的内部主机 A**。

可以看出，在内部主机与外部主机通信时，在NAT路由器上发生了两次地址转换：

- 离开专用网时：替换源地址，将内部地址替换为全球地址；
- 进入专用网时：替换目的地址，将全球地址替换为内部地址；

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20210502150326.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

