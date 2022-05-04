---
title: "Linux 下翻墙的终极方式"
date: 2022-05-04T16:19:40+08:00
tags: ["clash", "tproxy", "linux", "GFW", "fake-ip", "透明代理", "翻墙"]
image: iptables.webp
draft: false
---

本文展示了在Linux下使用clash和iptables搭建透明代理进行翻墙的步骤

我们用到了

* [clash](https://github.com/Dreamacro/clash) 作为代理程序
* [online subconverter](https://github.com/tindy2013/subconverter) 和 crontab 来定期更新配置文件
* [ACL4SSR](https://github.com/ACL4SSR/ACL4SSR/tree/master) 来提供GFW屏蔽域名规则

你可以从 [github gist](https://gist.github.com/cld4h/9a03ec2f826a25be5ab974fdbc540de4) 来下载本文中提到的文件。

## 参考

[clash-tproxy](https://www.sobyte.net/post/2022-02/clash-tproxy/)

[Transparent proxy demo code](https://git.breakpoint.cc/cgit/fw/tcprdr.git)

## 准备条件

* 安装好的 `clash` .(默认在 `/usr/bin/clash` 路径下)
* iptables
* systemd （你完全可以使用其他的启动程序，这不过在这里我只展示systemd的配置过程，因为大多数GNU/Linux发行版默认使用systemd）
* crontab

## 步骤

### 创建一个名为clash的系统用户

```
useradd -r -M -s /usr/sbin/nologin clash
```

由于我们要做全局的透明代理（代理网卡上的全部流量），所有我们需要避免代理软件发出的流量再次进入代理软件形成回路无限循环。

因此，我们创建一个独立的用户 `clash` (名字可以是任意的，但要一致) 来运行代理软件。
之后通过iptables的规则将 `clash` 用户的流量分离出来。

### 创建iptables和路由规则

创建一个内容是iptables命令的shell脚本 `/etc/clash/iptables.sh`.

文件包含三个主要部分：

1. iptables规则用于劫持DNS请求流量
2. PREROUTING 中 `clash` 链的配置
3. OUTPUT 中 `clash_local` 链的配置

iptables 示意图:

```txt
┌─────────┐                                                                                      ┌─────────┐
│         │             PREROUTING                                                INPUT          │         │
│         │                                                                                      │         │
│         │  ┌─────────────────────────────────────┐                      ┌────────────────────┐ │         │
│         │  │                                     │         /───\        │                    │ │         │
│         │  │ ┌───┐  ┌─────────┐  ┌──────┐  ┌───┐ │        /     \       │ ┌──────┐  ┌──────┐ │ │         │
│         ├──┤►│raw├─►│conntrack├─►│mangle├─►│nat├─┼──────►<routing>──────┤►│mangle├─►│filter├─┤►│         │
│         │  │ └───┘  └─────────┘  └──────┘  └───┘ │        \     /       │ └──────┘  └──────┘ │ │         │
│         │  │                                     │         \─┬─/        │                    │ │         │
│         │  └─────────────────────────────────────┘           │          └────────────────────┘ │         │
│         │                                                    │                                 │         │
│         │                               ┌──────────┐         │                                 │         │
│         │                               │          │         │                                 │         │
│         │                               │ ┌──────┐ │         │                                 │         │
│         │                               │ │mangle◄─┼─────────┘                                 │         │
│ Network │                               │ └──┬───┘ │                                           │  Local  │
│Interface│                               │    │     │ FORWARD                                   │ Process │
│         │                               │ ┌──▼───┐ │                                  /───\    │         │
│         │               ┌───────────────┼─┤filter│ │                                 /     \   │         │
│         │               │               │ └──────┘ │                                <routing>◄─┤         │
│         │               │               │          │                                 \     /   │         │
│         │               │               └──────────┘                                  \─┬─/    │         │
│         │               │                                                               │      │         │
│         │  ┌────────────┼────┐               ┌──────────────────────────────────────────┼────┐ │         │
│         │  │            │    │     ─────     │                                          │    │ │         │
│         │  │ ┌───┐  ┌───▼──┐ │    /     \    │ ┌──────┐  ┌───┐  ┌──────┐  ┌─────────┐  ┌▼──┐ │ │         │
│         │◄─┼─┤nat│◄─┤mangle│◄├───<reroute>───┼─┤filter│◄─┤nat│◄─┤mangle│◄─┤conntrack│◄─┤raw│ │ │         │
│         │  │ └───┘  └──────┘ │    \check/    │ └──────┘  └───┘  └──────┘  └─────────┘  └───┘ │ │         │
│         │  │                 │     ─────     │                                               │ │         │
│         │  └─────────────────┘               └───────────────────────────────────────────────┘ │         │
│         │     POSTROUTING                                        OUTPUT                        │         │
│         │                                                                                      │         │
└─────────┘                                                                                      └─────────┘
```

`/etc/clash/iptables.sh` 内容如下：

```
#!/usr/bin/env bash

# set -ex 选项说明:
# -e 当有命令以非0值退出时立刻结束执行
# -x 当命令执行时打印命令及其参数
set -ex

# ENABLE ipv4 forward
sysctl -w net.ipv4.ip_forward=1

# ROUTE RULES
ip rule add fwmark 666 table 666
ip route add local 0.0.0.0/0 table 666 dev lo

##################################
## hijack the dns query traffic ##
##################################

# 转发从局域网收到的所有 DNS 查询流量到 1053 端口
# 此操作会导致所有 DNS 请求全部返回虚假 IP(fake ip 198.18.0.1/16)
iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to 1053
# ipv6 同样配置
ip6tables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to 1053


# 转发从所有本机进程（除去uid为clash的进程）收到的所有 DNS 查询流量到 1053 端口
# 此操作会导致所有 DNS 请求全部返回虚假 IP(fake ip 198.18.0.1/16)
iptables -t nat -N clash_dns
iptables -t nat -A OUTPUT -p udp --dport 53 -j clash_dns
iptables -t nat -A clash_dns -m owner --uid-owner clash -j RETURN
iptables -t nat -A clash_dns -p udp -j REDIRECT --to-ports 1053
# ipv6 同样配置
ip6tables -t nat -N clash_dns
ip6tables -t nat -A OUTPUT -p udp --dport 53 -j clash_dns
ip6tables -t nat -A clash_dns -m owner --uid-owner clash -j RETURN
ip6tables -t nat -A clash_dns -p udp -j REDIRECT --to-ports 1053

##############################
## config for `clash` chain ##
##############################

# `clash` 链负责利用tproxy将流量转发到clash的7893端口
iptables -t mangle -N clash

# 目标地址为局域网或保留地址的流量跳过处理
# 保留地址参考:
# https://zh.wikipedia.org/wiki/%E5%B7%B2%E5%88%86%E9%85%8D%E7%9A%84/8_IPv4%E5%9C%B0%E5%9D%80%E5%9D%97%E5%88%97%E8%A1%A8
# https://zh.wikipedia.org/wiki/%E4%B8%93%E7%94%A8%E7%BD%91%E7%BB%9C
# IANA - Local Identification
iptables -t mangle -A clash -d 0.0.0.0/8 -j RETURN
# IANA - Loopback
iptables -t mangle -A clash -d 127.0.0.0/8 -j RETURN
# IANA - Private Use
iptables -t mangle -A clash -d 10.0.0.0/8 -j RETURN
iptables -t mangle -A clash -d 172.16.0.0/12 -j RETURN
iptables -t mangle -A clash -d 192.168.0.0/16 -j RETURN
# IPv4 Link-Local Addresses
iptables -t mangle -A clash -d 169.254.0.0/16 -j RETURN
# Multicast
iptables -t mangle -A clash -d 224.0.0.0/4 -j RETURN
# Future Use
iptables -t mangle -A clash -d 240.0.0.0/4 -j RETURN

# 其他所有流量转向到 7893 端口，并打上 mark
iptables -t mangle -A clash -p tcp -j TPROXY --on-port 7893 --tproxy-mark 666
iptables -t mangle -A clash -p udp -j TPROXY --on-port 7893 --tproxy-mark 666

# 将 `clash` 链加在PREROUTING链后面使其生效
iptables -t mangle -A PREROUTING -j clash

####################################
## config for `clash_local` chain ##
####################################

# `clash_local` 链负责处理本机的本地进程发出的流量：给发往公网的流量打上666标记
# (之后会reroute给lo设备，然后会在PREROUTING中的clash链上转发到tproxy 7893端口)
iptables -t mangle -N clash_local

# nerdctl 容器流量重新路由
#iptables -t mangle -A clash_local -i nerdctl2 -p udp -j MARK --set-mark 666
#iptables -t mangle -A clash_local -i nerdctl2 -p tcp -j MARK --set-mark 666

# 跳过内网流量
iptables -t mangle -A clash_local -d 0.0.0.0/8 -j RETURN
iptables -t mangle -A clash_local -d 127.0.0.0/8 -j RETURN
iptables -t mangle -A clash_local -d 10.0.0.0/8 -j RETURN
iptables -t mangle -A clash_local -d 172.16.0.0/12 -j RETURN
iptables -t mangle -A clash_local -d 192.168.0.0/16 -j RETURN
iptables -t mangle -A clash_local -d 169.254.0.0/16 -j RETURN
iptables -t mangle -A clash_local -d 224.0.0.0/4 -j RETURN
iptables -t mangle -A clash_local -d 240.0.0.0/4 -j RETURN

# 为本机发出的流量打 mark
# clash_local 链会为本机流量打 mark, 打过 mark 的流量会重新回到device lo，再从PREROUTING开始走流程.
iptables -t mangle -A clash_local -p tcp -j MARK --set-mark 666
iptables -t mangle -A clash_local -p udp -j MARK --set-mark 666

# 跳过来自服务端口的流量
# https web server
# iptables -t mangle -I OUTPUT -p tcp --sport 443 -j RETURN
# transmission
# iptables -t mangle -I OUTPUT -p tcp --sport 51413 -j RETURN

# 跳过 clash 程序本身发出的流量, 防止死循环(clash 程序需要使用 "clash" 用户启动)
iptables -t mangle -A OUTPUT -p tcp -m owner --uid-owner clash -j RETURN
iptables -t mangle -A OUTPUT -p udp -m owner --uid-owner clash -j RETURN

# 将 `clash_local` 链加在 OUTPUT 链后面使其生效
iptables -t mangle -A OUTPUT -j clash_local

#######################
## unimportant staff ##
#######################

# 修复 ICMP(ping)
# 这并不能保证 ping 结果有效(clash 等不支持转发 ICMP), 只是让它有返回结果而已
# --to-destination 设置为一个可达的地址即可
#sysctl -w net.ipv4.conf.all.route_localnet=1
#iptables -t nat -A PREROUTING -p icmp -d 198.18.0.0/16 -j DNAT --to-destination 127.0.0.1
#iptables -t nat -A OUTPUT -p icmp -d 198.18.0.0/16 -j DNAT --to-destination 127.0.0.1
```

当执行完上述脚本后，iptables的状态如下图所示：

```txt
┌─────────┐                                                                                      ┌─────────┐
│         │             PREROUTING           rule1*                               INPUT          │         │
│         │                                    │                                                 │         │
│         │  ┌─────────────────────────────────┼───┐                      ┌────────────────────┐ │         │
│         │  │                                 │   │         /───\        │                    │ │         │
│         │  │ ┌───┐  ┌─────────┐  ┌──────┐  ┌─┴─┐ │        /     \       │ ┌──────┐  ┌──────┐ │ │         │
│         ├──┤►│raw├─►│conntrack├─►│mangle├─►│nat├─┼──────►<routing>──────┤►│mangle├─►│filter├─┤►│         │
│         │  │ └───┘  └─────────┘  └┬─────┘  └───┘ │        \     /       │ └──────┘  └──────┘ │ │         │
│         │  │                      │              │         \─┬─/        │                    │ │         │
│         │  └──────────────────────┼──────────────┘           │          └────────────────────┘ │         │
│         │                         │                          │                                 │         │
│         │                      ┌──┴──┐  ┌──────────┐         │                                 │         │
│         │                      │clash│  │          │         │                                 │         │
│         │                      └─────┘  │ ┌──────┐ │         │                                 │         │
│         │                               │ │mangle◄─┼─────────┘                                 │         │
│ Network │                               │ └──┬───┘ │              ┌───────────┐                │  Local  │
│Interface│                               │    │     │ FORWARD      │clash_local│                │ Process │
│         │                               │ ┌──▼───┐ │              └─┬─────────┘       /───\    │         │
│         │               ┌───────────────┼─┤filter│ │                │                /     \   │         │
│         │               │               │ └──────┘ │                ├───rule3*      <routing>◄─┤         │
│         │               │               │          │                │                \     /   │         │
│         │               │               └──────────┘                ├───rule2*        \─┬─/    │         │
│         │               │                                           │                   │      │         │
│         │  ┌────────────┼────┐               ┌──────────────────────┼───────────────────┼────┐ │         │
│         │  │            │    │     ─────     │                      │                   │    │ │         │
│         │  │ ┌───┐  ┌───▼──┐ │    /     \    │ ┌──────┐  ┌───┐  ┌───┴──┐  ┌─────────┐  ┌▼──┐ │ │         │
│         │◄─┼─┤nat│◄─┤mangle│◄├───<reroute>───┼─┤filter│◄─┤nat│◄─┤mangle│◄─┤conntrack│◄─┤raw│ │ │         │
│         │  │ └───┘  └──────┘ │    \check/    │ └──────┘  └┬──┘  └──────┘  └─────────┘  └───┘ │ │         │
│         │  │                 │     ─────     │            │                                  │ │         │
│         │  └─────────────────┘               └────────────┼──────────────────────────────────┘ │         │
│         │     POSTROUTING                                 │      OUTPUT                        │         │
│         │                                          ┌──────┴──┐                                 │         │
└─────────┘                                          │clash_dns│                                 └─────────┘
                                                     └─────────┘

 *rule1: iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to 1053
 *rule2: iptables -t mangle -A OUTPUT -p tcp -m owner --uid-owner clash -j RETURN
 *rule3: iptables -t mangle -A OUTPUT -p udp -m owner --uid-owner clash -j RETURN
```

研究一下 `iptables.sh` 中的路由规则:

```sh
ip rule add fwmark 666 table 666
ip route add local 0.0.0.0/0 table 666 dev lo
```

`ip rule` 用于控制策略路由数据库，管理策略路由的路由选择算法

这里 `ip rule` 命令让内核对标记有666的数据包查找666号路由表

[ip rule 参考文档](http://linux-ip.net/html/tools-ip-rule.html)

`ip route` 命令在666号路由表中添加了一条到lo设备的默认路由

我们可以通过 `ip route show table 666` 来查看 `ip route add local 0.0.0.0/0 table 666 dev lo` 添加的路由项

查看路由表编号和名称的映射：
`cat /etc/iproute2/rt_tables`

 ### 创建规则清理脚本

在退出clash后需清除设定的规则

`-D` 或 `--delete`:

从所选链中删掉一条或多条规则。
可以指定要删除的规则号；或指定规则内容（将 `-A`、`-I` 替换成 `-D` 即可）

`-F` or `--flush`:

刷新所选链，相当于逐条清除链上的所有规则

`-X` or `--delete-chain`:

删除用户定义的链

```sh
#!/usr/bin/env bash

set -ex

ip rule del fwmark 666 table 666 || true
ip route del local 0.0.0.0/0 table 666 dev lo || true

iptables -t nat -D PREROUTING -p udp --dport 53 -j REDIRECT --to 1053 || true
ip6tables -t nat -D PREROUTING -p udp --dport 53 -j REDIRECT --to 1053 || true
iptables -t nat -D OUTPUT -p udp --dport 53 -j clash_dns || true
ip6tables -t nat -D OUTPUT -p udp --dport 53 -j clash_dns || true
iptables -t nat -F clash_dns || true
iptables -t nat -X clash_dns || true
ip6tables -t nat -F clash_dns || true
ip6tables -t nat -X clash_dns || true
#iptables -t nat -D PREROUTING -p icmp -d 198.18.0.0/16 -j DNAT --to-destination 127.0.0.1
#iptables -t nat -D OUTPUT -p icmp -d 198.18.0.0/16 -j DNAT --to-destination 127.0.0.1

iptables -t mangle -D PREROUTING -j clash || true
iptables -t mangle -F clash || true
iptables -t mangle -X clash || true

iptables -t mangle -D OUTPUT -p tcp -m owner --uid-owner clash -j RETURN || true
iptables -t mangle -D OUTPUT -p udp -m owner --uid-owner clash -j RETURN || true
iptables -t mangle -D OUTPUT -j clash_local || true
iptables -t mangle -F clash_local || true
iptables -t mangle -X clash_local || true
```

### 修正ping

fake-ip 模式的一个主要缺点是无法ping

前面 `iptables.sh` 中将所有ping请求重定向到127.0.0.1 的操作是掩耳盗铃的，它不提供任何信息。

但是，可以通过clash用户的身份进行ping

```sh
alias ping="sudo -u clash ping"
```

### 提供systemd unit file 以一键启动clash及相关iptables等配置

此步骤是可选的，但我觉得能够一键启动/关闭clash很方便。

同时也可以设置clash开机启动。

Systemd 配置文件命名为`clash.service` ，放在 `/usr/lib/systemd/system` 目录下，执行 `sudo systemctl start clash` 来启动运行clash

开机启动： `sudo systemctl enable clash`.

### 配置clash

下面看一看对clash本身的配置

clash 使用 `-d` 选项得到配置文件目录； `-f` 选项得到配置文件

当 `-f` 被忽略时，默认的配置文件名是 `config.yaml`

我们想要通过 `crontab` 从clash的订阅链接中定时更新clash的配置文件

#### 构造订阅连接

TL;DR. 将下面URL中`XXXXXXXX` 替换成[URL Encode后的](https://www.urlencoder.io/)节点订阅URL即可

```
https://sub.xeton.dev/sub?target=clash&new_name=true&filename=config.yaml&url=XXXXXXXX&config=https%3A%2F%2Fgist.githubusercontent.com%2Fcld4h%2F9a03ec2f826a25be5ab974fdbc540de4%2Fraw%2Fconfig.ini
```

我们通过[subconverter](https://github.com/tindy2013/subconverter)来生成clash的配置文件。

你需要修改订阅链接中的一些参数以使得生成的clash正确配置成 **监听tproxy** 以及 **fake-ip** 模式

具体的：

1. `url` 参数要是正确的节点信息并且经过[URL Encode](https://www.urlencoder.io/).
2. 多个节点订阅URL要用 `|` 分割, 此符号[URL Encode](https://www.urlencoder.io/) 之后是 `%7C` .
3. `config` 参数是用于告诉subconverter如何进行订阅转换的关键，应当[配置正确](https://gist.githubusercontent.com/cld4h/9a03ec2f826a25be5ab974fdbc540de4/raw/config.ini). ⚠️

⚠️ 注意: 这里subconverter 的配置文件非常重要

[ACL4SSR 项目提供了一个默认的subconverter配置文件](https://raw.githubusercontent.com/ACL4SSR/ACL4SSR/master/Clash/config/ACL4SSR_Online.ini
)，这个文件没问题，但默认的选项不足以让clash开启fake-ip模式并监听tproxy

要想使其工作，需指定 `clash_rule_base`： 在[其](https://raw.githubusercontent.com/ACL4SSR/ACL4SSR/master/Clash/config/ACL4SSR_Online.ini)中插入下面这一行

```
clash_rule_base=https://gist.githubusercontent.com/cld4h/9a03ec2f826a25be5ab974fdbc540de4/raw/clash-base-config.yaml
```

这一行的作用是告诉subconverter使用[URL中的文件](https://gist.githubusercontent.com/cld4h/9a03ec2f826a25be5ab974fdbc540de4/raw/clash-base-config.yaml)作为clash的基础配置

或者你可以直接使用[我的config.ini](https://gist.githubusercontent.com/cld4h/9a03ec2f826a25be5ab974fdbc540de4/raw/config.ini)

示例URL:

```txt
https://sub.xeton.dev/sub?        --- 后端地址
target=clash                      --- 客户端类型
&new_name=true                    --- 重命名
&filename=config.yaml             --- 文件名
&url=                             --- 节点信息
ss%3A%2F%2FYmYtY2ZiOnRlc3QvIUAjOkAxOTIuMTY4LjEwMC4xOjg4ODg%23example-server
%7C                               --- 分隔符 |
ss%3A%2F%2FYmYtY2ZiOnRlc3QvIUAjOkAxOTIuMTY4LjEwMC4xOjg4ODg%23example-server-2
&config=                          --- subconverter的配置文件; .ini 格式
https%3A%2F%2Fgist.githubusercontent.com%2Fcld4h%2F9a03ec2f826a25be5ab974fdbc540de4%2Fraw%2Fconfig.ini
                                  --- 下面这些配置是多余的，没啥用
&list=false                       --- 用于输出 Surge Node List 或者 Clash Proxy Provider 或者 Quantumult (X) 的节点订阅 或者 解码后的 SIP002
&tfo=false                        --- 用于开启该订阅链接的 TCP Fast Open，默认为 false
&scv=false                        --- 用于关闭 TLS 节点的证书检查，默认为 false
&fdn=false                        --- 用于过滤目标类型不支持的节点，默认为 true
&sort=false                       --- 用于对输出的节点或策略组按节点名进行再次排序，默认为 false
```
#### 设置定时更新订阅任务

创建订阅更新脚本 `/etc/clash/update-config.sh` ，内容如下：

```sh
#!/usr/bin/env bash
set -ex
/usr/bin/curl "https://sub.xeton.dev/sub?target=clash&new_name=true&filename=config.yaml&url=XXXXXXXX&config=https%3A%2F%2Fgist.githubusercontent.com%2Fcld4h%2F9a03ec2f826a25be5ab974fdbc540de4%2Fraw%2Fconfig.ini" -o /etc/clash/config.yaml
```

⚠️ 上面的URL要替换一下

手动执行一下上述脚本确认配置文件能够正常生成。

确保你的clash`config.yaml`配置文件中有如下内容：

```yaml
tproxy-port: 7893

dns:
  enable: true
  listen: 0.0.0.0:1053
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
```

添加[定时任务](https://crontab.guru)： 执行 `sudo crontab -e` 并加入下面这一行以在每天早上10:00执行订阅更新脚本。

```sh
0 10 * * * /usr/bin/bash /etc/clash/update-config.sh
```
