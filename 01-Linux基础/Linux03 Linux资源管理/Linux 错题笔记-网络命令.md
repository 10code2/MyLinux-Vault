---
title: Linux 错题笔记-网络命令
created: 2026-10-14
updated: 2026-10-24
status: 待复习
tags: [linux, 基础]
---



> 分类：Linux网络 | 面试高频 | WSL排障

## 核心结论

1. `ip addr` 和 `ip a` **完全等价**，`a` 是 `addr` 的简写，iproute2工具集命令。
2. `ifconfig` 属于老旧 `net-tools` 工具，已经被iproute2淘汰；新版最小化系统（Debian/WSL）默认不带。
3. 现代运维、面试**优先使用 iproute2 系列命令**。

## 命令对照表（新旧替换）

|老命令（net-tools）|新命令（iproute2）|功能|
|---|---|---|
|ifconfig|ip addr / ip link|网卡IP、链路状态|
|route|ip route|路由表|
|arp|ip neigh|ARP邻居表|
|netstat|ss|端口与连接|

## 各命令说明

### ifconfig

- 安装：`apt install net-tools`
- 特点：
    - 默认只输出UP状态网卡，DOWN网卡不展示
    - 单网卡多IP场景，老版本容易丢失附加IP
    - 不支持VLAN、bond、vxlan、网络命名空间等高级网络
- 示例

```
ifconfig eth0 192.168.1.10 netmask 255.255.255.0
ifconfig eth0 up
ifconfig eth0 down
```

> ⚠️ 所有配置为临时生效，重启网络/服务器丢失。

### ip addr（简写 ip a）

> 职责：**三层IP地址管理**，**不能开关网卡**

```
ip addr add 192.168.1.10/24 dev eth0    # 添加IP
ip addr del 192.168.1.10/24 dev eth0    # 删除单个IP
ip addr flush dev eth0                  # 清空网卡全部IP
```

### ip link（简写 ip l）

> 职责：**二层链路管理**，网卡开关、MAC、MTU

```
ip link set eth0 up
ip link set eth0 down
ip link set eth0 mtu 1400
```

> ❗易错：不能用 `ip addr` 执行网卡up/down。

### 其他iproute2高频子命令

```
ip r                # ip route，查看路由
ip neigh            # ARP邻居表
ip netns            # 网络命名空间（容器底层）
```

## ip a 输出关键字段

```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP
    link/ether xx:xx:xx:xx:xx:xx        # MAC地址
    inet 172.29.139.48/20               # IPv4 + 掩码
    scope global                        # global=全网可达；link=仅本地链路
```

## 高频错题&面试坑点

1. ❌ `ip addr set eth0 up` ✅ 错误。网卡启停属于二层链路操作，必须用 `ip link set eth0 up`
2. Q：ifconfig看不到关闭的网卡怎么办？ ✅ 使用 `ip a`，UP/DOWN网卡全部列出，网络排障首选
3. Q：`ip a` 和 `ip addr` 是两个不同命令？ ❌ 否，`a`只是简写，功能完全一致
4. net-tools与iproute2可共存，但生产环境统一推荐iproute2

## 实操练习题（WSL可直接执行）

```
ip a
ip l
ip r
ip addr add 10.99.99.99/32 dev lo
ip addr del 10.99.99.99/32 dev lo
```

---

## 🔗 相关笔记

- [[Linux01 命令系统实战笔记]]
- [[Linux常见故障排障方法]]
- [[运维基础概念]]
