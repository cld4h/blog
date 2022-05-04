---
title: "The Ultimate Way to Bypass GFW on Linux"
date: 2022-05-04T16:19:40+08:00
tags: ["clash", "tproxy", "linux", "GFW", "fake-ip", "透明代理", "翻墙"]
image: iptables.webp
draft: false
---

This article show you the ultimate way to set up a transparent proxy on Linux using clash and iptables to bypass the GFW in China.

We use:

* [clash](https://github.com/Dreamacro/clash) as the proxy software
* [online subconverter](https://github.com/tindy2013/subconverter) and crontab to periodically update config file.
* [ACL4SSR](https://github.com/ACL4SSR/ACL4SSR/tree/master) to provide GFW rules.

You can go to [github gist](https://gist.github.com/cld4h/9a03ec2f826a25be5ab974fdbc540de4) to download all files mentioned in this article.

## Reference

[clash-tproxy](https://www.sobyte.net/post/2022-02/clash-tproxy/)

[Transparent proxy demo code](https://git.breakpoint.cc/cgit/fw/tcprdr.git)

## Prerequisite

* a `clash` installed. (By default in `/usr/bin/clash`)
* iptables
* systemd (It's totally fine to use other init programs, but here I'll only show the configuration of systemd, which is the default of most GNU/Linux distros.)
* crontab


## Steps

### Create a system user called clash

```
useradd -r -M -s /usr/sbin/nologin clash
```

Since we want to proxy all internet traffics, we need to prevent the traffic from looping.
Thus, we create a distinct user called `clash` (the name can be arbitrary) to run the proxy software `clash`.
We'll make an iptables rule in the following steps to distinguish traffic coming from the user `clash`.

### Create iptables and routing rules

Create a shell script of iptables command called `/etc/clash/iptables.sh`.

The file contains 3 main sections:

1. iptables rules to hijack the dns query traffic,
2. config for `clash` chain
3. config for `clash_local` chain

iptables:

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

`/etc/clash/iptables.sh`

```
#!/usr/bin/env bash

# set -ex options:
# -e Exit immediately if a command exits with a non-zero status
# -x Print commands and their arguments as they are executed.
set -ex

# ENABLE ipv4 forward
sysctl -w net.ipv4.ip_forward=1

# ROUTE RULES
ip rule add fwmark 666 table 666
ip route add local 0.0.0.0/0 table 666 dev lo

##################################
## hijack the dns query traffic ##
##################################

# Redirect all dns query traffic from LAN to port 1053
# Later clash will return a fake ip in 198.18.0.1/16
iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to 1053
# same for ipv6
ip6tables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to 1053


# Redirect dns query traffic from all local processes (except owned by user clash) to port 1053
# Later clash will return a fake ip in 198.18.0.1/16
iptables -t nat -N clash_dns
iptables -t nat -A OUTPUT -p udp --dport 53 -j clash_dns
iptables -t nat -A clash_dns -m owner --uid-owner clash -j RETURN
iptables -t nat -A clash_dns -p udp -j REDIRECT --to-ports 1053
# same for ipv6
ip6tables -t nat -N clash_dns
ip6tables -t nat -A OUTPUT -p udp --dport 53 -j clash_dns
ip6tables -t nat -A clash_dns -m owner --uid-owner clash -j RETURN
ip6tables -t nat -A clash_dns -p udp -j REDIRECT --to-ports 1053

##############################
## config for `clash` chain ##
##############################

# `clash` chain for using tproxy to redirect traffic to the clash listening port 7893
iptables -t mangle -N clash

# skip traffic to LAN or reserved address
# reference:
# https://en.wikipedia.org/wiki/List_of_assigned_/8_IPv4_address_blocks
# https://en.wikipedia.org/wiki/Private_network#IPv4
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

# Forward all the other traffic to port 7893 and set tproxy mark
iptables -t mangle -A clash -p tcp -j TPROXY --on-port 7893 --tproxy-mark 666
iptables -t mangle -A clash -p udp -j TPROXY --on-port 7893 --tproxy-mark 666

# Append the `clash` chain to PREROUTING to enable it
iptables -t mangle -A PREROUTING -j clash

####################################
## config for `clash_local` chain ##
####################################

# `clash_local` chain to manipulate the traffic from local process: set fwmark 666 on traffic to public address
# (which will later reroute to dev lo, then redirect to tproxy port 7893 on the `clash` chain of `PREROUTING`)
iptables -t mangle -N clash_local

# rerouting traffic from nerdctl container
#iptables -t mangle -A clash_local -i nerdctl2 -p udp -j MARK --set-mark 666
#iptables -t mangle -A clash_local -i nerdctl2 -p tcp -j MARK --set-mark 666

# Don't touch traffic to LAN and reserved address
iptables -t mangle -A clash_local -d 0.0.0.0/8 -j RETURN
iptables -t mangle -A clash_local -d 127.0.0.0/8 -j RETURN
iptables -t mangle -A clash_local -d 10.0.0.0/8 -j RETURN
iptables -t mangle -A clash_local -d 172.16.0.0/12 -j RETURN
iptables -t mangle -A clash_local -d 192.168.0.0/16 -j RETURN
iptables -t mangle -A clash_local -d 169.254.0.0/16 -j RETURN
iptables -t mangle -A clash_local -d 224.0.0.0/4 -j RETURN
iptables -t mangle -A clash_local -d 240.0.0.0/4 -j RETURN

# set mark for local traffic
# clash_local set mark for the traffic from local process,
# the marked traffic will come back to PREROUTING chain.
iptables -t mangle -A clash_local -p tcp -j MARK --set-mark 666
iptables -t mangle -A clash_local -p udp -j MARK --set-mark 666

# Don't touch traffic from a serving port
# https web server
# iptables -t mangle -I OUTPUT -p tcp --sport 443 -j RETURN
# transmission
# iptables -t mangle -I OUTPUT -p tcp --sport 51413 -j RETURN

# Don't touch traffic from the user `clash` to prevent looping
iptables -t mangle -A OUTPUT -p tcp -m owner --uid-owner clash -j RETURN
iptables -t mangle -A OUTPUT -p udp -m owner --uid-owner clash -j RETURN

# Append the `clash_local` chain to OUTPUT to enable it
iptables -t mangle -A OUTPUT -j clash_local

#######################
## unimportant staff ##
#######################

# fix ICMP(ping)
# This step does NOT make the ping result validate
# (clash doesn't support forwarding ICMP), it just makes ping have a result.
# set --to-destination to a accessable address.
#sysctl -w net.ipv4.conf.all.route_localnet=1
#iptables -t nat -A PREROUTING -p icmp -d 198.18.0.0/16 -j DNAT --to-destination 127.0.0.1
#iptables -t nat -A OUTPUT -p icmp -d 198.18.0.0/16 -j DNAT --to-destination 127.0.0.1
```

After applying the script above, the system iptables should be like this:

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

It's also worth to take a look at the route rules in `iptables.sh`:

```sh
ip rule add fwmark 666 table 666
ip route add local 0.0.0.0/0 table 666 dev lo
```

`ip rule` manipulates rules in the routing policy database control the route selection algorithm.

So here the `ip rule` command make the kernel to lookup table 666 for packets with fwmark 666

[ip rule reference is here.](http://linux-ip.net/html/tools-ip-rule.html)

The `ip route` command add a default route to device lo on routing table 666.

We can use `ip route show table 666` to see the route added by `ip route add local 0.0.0.0/0 table 666 dev lo
`

See the route table value and name mapping:
`cat /etc/iproute2/rt_tables`



 ### Create a cleaning script

Create a shell script to clean the iptables.

`-D` or `--delete`:

Delete one or more rules from the selected chain.
There are two versions of this command:
the rule can be specified as a number in the chain (starting at 1 for the first rule) or a rule to match.

`-F` or `--flush`:

Flush the selected chain (all the chains in the table if none is given).
This is equivalent to deleting all the rules one by one.

`-X` or `--delete-chain`:

Delete the optional user-defined chain specified.
There must be no references to the chain.
If there are, you must delete or replace the referring rules before the chain can be deleted.
The chain must be empty, i.e. not contain any rules.
If no argument is given, it will attempt to delete every non-builtin chain in the table.

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

### Fixing ping

One major drawback of the fake-ip mode is that you can't ping a fake ip.

The solution above to redirect all ping request to the localhost is "掩耳盗铃", it doesn't provide any useful information.

Sadly, all devices from LAN can't use ping, but fortunately you can alias ping to `sudo -u clash ping` for this gateway to use ping.

```sh
alias ping="sudo -u clash ping"
```

### Provide a systemd unit file to start clash at a single command

This step is optional, but I felt it's extremely convenient to start clash by running a single command.

You can also make clash start at boot time.

The systemd config file is in the appended files with the name `clash.service`

put it under `/usr/lib/systemd/system`, and run `sudo systemctl start clash` to start running clash.

To start it at boot time, run `sudo systemctl enable clash`.

### Configuring clash

Now let's set clash up to work.

Clash use `-d` option to set configuration directory and `-f` option to specify configuration file.

The default configuration file name is `config.yaml` when `-f` is omitted.

We want to use `crontab` to periodically update the clash config from a subscription url.

#### Construct the subscription URL

TL;DR. Just replace `XXXXXXXX` with your [encoded](https://www.urlencoder.io/) subscription URL .

```
https://sub.xeton.dev/sub?target=clash&new_name=true&filename=config.yaml&url=XXXXXXXX&config=https%3A%2F%2Fgist.githubusercontent.com%2Fcld4h%2F9a03ec2f826a25be5ab974fdbc540de4%2Fraw%2Fconfig.ini
```

We use [subconverter](https://github.com/tindy2013/subconverter) to generate config file for clash.

You need to change a few parameters in the subscription URL
to generate a proper config file for clash to work in fake-ip mode.

To be specific:

1. the `url` parameter should be the correct subscription URL and [encoded](https://www.urlencoder.io/).
2. Multiple subscription URL should be seperated by `|`, or `%7C` after [URL Encoding](https://www.urlencoder.io/).
3. the `config` parameter should point to [a correct configuration file](https://gist.githubusercontent.com/cld4h/9a03ec2f826a25be5ab974fdbc540de4/raw/config.ini). ⚠️

⚠️ NOTICE: the configuration file for subconverter is extremely important!

[ACL4SSR project provide a default configuration file for subconverter](https://raw.githubusercontent.com/ACL4SSR/ACL4SSR/master/Clash/config/ACL4SSR_Online.ini
), it's fine but NOT ENOUGH for clash to work on fake-ip mode.

To make it work, you need specify `clash_rule_base` by inserting this line to [the configuration file for **subconverter**](https://raw.githubusercontent.com/ACL4SSR/ACL4SSR/master/Clash/config/ACL4SSR_Online.ini)

```
clash_rule_base=https://gist.githubusercontent.com/cld4h/9a03ec2f826a25be5ab974fdbc540de4/raw/clash-base-config.yaml
```

What this line does is that it tells the subconverter to use [the file in the url above](https://gist.githubusercontent.com/cld4h/9a03ec2f826a25be5ab974fdbc540de4/raw/clash-base-config.yaml) instead of the default clash base configuration.

Or you can just use [the config.ini here](https://gist.githubusercontent.com/cld4h/9a03ec2f826a25be5ab974fdbc540de4/raw/config.ini)

Sample url:

```txt
https://sub.xeton.dev/sub?        --- backend address
target=clash                      --- client type
&new_name=true                    --- create a new filename
&filename=config.yaml             --- filename
&url=                             --- Node info
ss%3A%2F%2FYmYtY2ZiOnRlc3QvIUAjOkAxOTIuMTY4LjEwMC4xOjg4ODg%23example-server
%7C                               --- the delimiter | to seperate multiple urls
ss%3A%2F%2FYmYtY2ZiOnRlc3QvIUAjOkAxOTIuMTY4LjEwMC4xOjg4ODg%23example-server-2
&config=                          --- config file for subconverter; .ini format
https%3A%2F%2Fgist.githubusercontent.com%2Fcld4h%2F9a03ec2f826a25be5ab974fdbc540de4%2Fraw%2Fconfig.ini
                                  --- options below is unnecessary and can be omitted
&list=false                       --- 用于输出 Surge Node List 或者 Clash Proxy Provider 或者 Quantumult (X) 的节点订阅 或者 解码后的 SIP002
&tfo=false                        --- 用于开启该订阅链接的 TCP Fast Open，默认为 false
&scv=false                        --- 用于关闭 TLS 节点的证书检查，默认为 false
&fdn=false                        --- 用于过滤目标类型不支持的节点，默认为 true
&sort=false                       --- 用于对输出的节点或策略组按节点名进行再次排序，默认为 false
```
#### Set up a cronjob

Create a shell script `/etc/clash/update-config.sh` with the following content:

```sh
#!/usr/bin/env bash
set -ex
/usr/bin/curl "https://sub.xeton.dev/sub?target=clash&new_name=true&filename=config.yaml&url=XXXXXXXX&config=https%3A%2F%2Fgist.githubusercontent.com%2Fcld4h%2F9a03ec2f826a25be5ab974fdbc540de4%2Fraw%2Fconfig.ini" -o /etc/clash/config.yaml
```

⚠️ The URL should be replace.

Manually executing it to make sure the config file is generated successfully.

Make sure you have the following contents in your `config.yaml` file for clash:

```yaml
tproxy-port: 7893

dns:
  enable: true
  listen: 0.0.0.0:1053
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
```

Add a [cronjob](https://crontab.guru) by executing `sudo crontab -e` and add this line to executing the script at 10:00 AM every day.

```sh
0 10 * * * /usr/bin/bash /etc/clash/update-config.sh
```
