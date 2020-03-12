

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [1. WireGuard简介](#1-wireguard简介)
  - [1.1. 特性](#11-特性)
- [2. 概述](#2-概述)
  - [2.1 简单的网络接口](#21-简单的网络接口)
  - [2.2 加密密钥路由](#22-加密密钥路由)
  - [2.3 安装WireGuard](#23-安装wireguard)

<!-- /code_chunk_output -->


# 1. WireGuard简介

WireGuard 是一个广受业界关注的，并具有简单、快速、现代化等特性并采用了最先进加密技术的的轻量级VPN工具。它的设计目标是比IPSec更快、更精简、更具有可用性，同时避免了令人头痛的复杂配置。同时WireGuard比OpenVPN性能更高更具有通用性，可以广泛的部署在嵌入式设备到超级计算机上。最初WireGuard是为Linux内核开发的，目前已经成为了跨平台软件（支持Windows、macOS、BDS、iOS、Android）。目前，WireGuard正在活跃开发中，但已经被认为是业界最安全、最易用和最简单的VPN解决方式。

## 1.1. 特性

1. 配置简单
WireGuard的设计目标是和SSH协议一样简单易用。VPN只需简单的交换公私钥即可建立连接（和SSH密钥交换完全一样）。同时，它还支持IP漫游（类似于mosh）。配置完成后，WireGuard是透明的，并提供了一个非常基础，但功能强大的接口。

1. 加密功能强大
WireGuard采用了最先进的加密技术，如Noise协议框架、Curve25519、ChaCha20、Poly1305、BLAKE2、SipHash24、HKDF和安全可信结构。它采用了合理且保守的选择，并已经被密码学专家审查。

1. 更小的攻击免
WireGuard设计之初便具有易于实施和简单性。它旨在采用极少数代码便可实现功能，并且可以很方便的审核安全漏洞。就像*Swan/IPsec 和 OpenVPN/OpenSSL这类庞然大物，审查代码对于一个大型的安全团队来说也是一个艰巨的任务，而WireGuard精简代码，单个个人便可对其全面审查。

1. 高性能
极高的加解密速度加上WireGuard和Linux内核进行深度结合，意味着更高的网络速度。它可以适用于小型嵌入式设备，比如智能手机和满载的骨干路由。

# 2. 概述
本文不对代码细节做说明，如果有代码上的任何疑问，可以阅读官方的[产品白皮书](https://www.wireguard.com/papers/wireguard.pdf)，上面详细介绍了协议、密码学和基础设计理念。

WireGuard通过UDP加密封装IP数据包。密钥的分发和配置推送均不在WireGuard的软件范围内，以避免出现像IKE或OpenVPN哪有的功能膨胀。它更像SSH那样，双方都有对方的公钥，然后只需通过连接交换数据包。

## 2.1 简单的网络接口
WireGuard的工作原理是，添加一个type为wireguard的网络接口，可以为这个接口配置IP地址、添加或删除路由。使用[`wg(8)`](https://git.zx2c4.com/wireguard-tools/about/src/man/wg.8)工具配置wireguard的其他选项，将此接口作为隧道。

WireGuard将隧道IP地址和远程端口相关联，当接口（interface）发送数据到对端（peer）时，进行如下操作:

1. 判断该数据包属于IP：192.168.30.8，判断是否属于peer组，是则进行步骤2，否则丢弃；
1. 使用对应peer组的公钥加密数据包；
1. 获取对应peer组的远程IP和端口（Endpoint），例如Endpoint=216.58.211.110:53133；
1. 最后使用UDP将步骤2所加密的封装的报文通过Internet发送到216.58.211.110:53133；



当接口（interface）收到数据包时，会进行如下操作：

1. 收到从远端主机的某个UDP端口发过来的数据，比如收到从98.139.183.24的UDP 7361端口发过来的数据，然后使用私钥进行解密；
1. 将验证数据发送至来源端口，当对端成功使用自身私钥解密后，更新来源IP端口（98.139.183.24:7631）为新的Endpoint；
1. 解密后，纯文本数据包来自192.168.43.89，根据配置判断是否允许对端以192.168.43.89形式向本机发送数据包；
1. 是则接受数据包，否则丢弃；

## 2.2 加密密钥路由
WireGuard的核心是被称为`Cryptokey Routing`的概念，他的工作原理是将公钥与隧道内允许的IP地址相关联，每个接口（interface）都有一个私钥匙和对端（peer）列表，每个对端（peer）都有一个公钥。公钥长度短且简单，由对等连接相互验证，配置可以在任何带外方法在外部传递，类似于将ssh公钥发送给朋友，用于远程连接他服务器shell的方式：
例如，WireGuard的服务器（在wg的概念中，没有严格的服务器和客户端的区别，在这里以用于区分角色）端有如下配置：
```
[Interface]
PrivateKey = yAnz5TF+lXXJte14tji3zlMNq+hd2rYUIgJBgB3fBmk=
ListenPort = 51820

[Peer]
PublicKey = xTIBA5rboUvnH4htodjb6e697QjLERt1NAB4mZqp8Dg=
AllowedIPs = 10.192.122.3/32, 10.192.124.1/24

[Peer]
PublicKey = TrMvSoP4jYQlY6RIzBgbssQqY3vxI2Pi+y71lOWWXX0=
AllowedIPs = 10.192.122.4/32, 192.168.0.0/16

[Peer]
PublicKey = gN65BkIKy1eCE9pP1wdc8ROUtkHLF2PfAqYdyYBz6EA=
AllowedIPs = 10.10.10.230/32
```

客户端的配置如下：
```
[Interface]
PrivateKey = gI6EdUSYvn8ugXOt8QQD6Yc+JyiZxIhp3GInSWRfWGE=
ListenPort = 21841

[Peer]
PublicKey = HIgo9xNzJMWLKASShiTqIybxZ0U3wGLiUeJ1PKf8ykw=
Endpoint = 192.95.5.69:51820
AllowedIPs = 0.0.0.0/0
```
在服务器配置中，每个`Peer`（客户端）能够将数据发送到`Interface`，其源IP匹配其相应的允许IP列表。例如：当服务器在解密和验证后从对等方接受的数据包时，如果其源IP为10.10.10.230，则允许该数据进入接口；否则，将数据丢弃；`gN65BkIK...`

在服务器配置中，当`Interface`希望将数据发送到`Peer`（客户端）时，它将查看数据包的目标IP，并将IP与每个允许IP列表进行比较，以查看最终将数据发送到哪个`Peer`，例如：当向目标IP为10.10.10.230的数据包发送到`Peer`时，他将使用匹配的`Peer`公钥`gN65BkIK...`进行加密，然后发送到该`Peer`的最新Internet entpoint上；

在客户端中，允许的IP设置为了0.0.0.0/0，所以，所有源IP都会接受，并进行解密验证，同样，所有目标IP都会`Peer`的公钥`HIgo9xNz...`加密，并通过internet发送给服务器的`Endpoint`

简而言之，在发送数据时，允许IP列表`AllowedIPs`作为路由表而使用，当接受数据时，`AllowedIPS`则充当了一种访问控制列表

这就是WireGuard所说的加密密钥路由表：公钥简单的关联允许的IP。

`AllowedsIPS`可以自由组合IPv4或IPv6，必要时，一种IP完全可以封装在另一种IP内发送

由于在 WireGuard 接口上发送的所有数据包都经过加密和身份验证，并且由于对等方的标识和对等方的允许 IP 地址之间存在如此紧密的耦合，因此系统管理员不需要复杂的防火墙扩展，例如，在IPsec的情况下，但他们可以简单地匹配"是从这个IP？在此接口上？，并保证它是一个安全和真实的数据包。这大大简化了网络管理和访问控制，并提供了更多的保证，您的 iptables 规则实际上在做您打算做的事情。


## 2.3 安装WireGuard
安装WireGuard的方式在[官方文档](https://www.wireguard.com/install/)内有详细介绍，这里不再过多说明

