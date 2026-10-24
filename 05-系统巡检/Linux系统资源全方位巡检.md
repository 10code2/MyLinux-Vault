---
title: Linux系统资源全方位巡检
created: 2026-10-14
updated: 2026-10-24
status: 待复习
tags: [巡检, 性能]
---



# 笔记1：Linux 系统资源全方位查看（CPU/内存/磁盘/进程/网络）

## 简介

本篇为 **标准系统巡检笔记** 所有命令作用：**查看状态、采集数据、观察资源** 不涉及排障、不涉及原理深挖，纯「运维日常巡检手册」

# 一、CPU 资源查看

## 1. 实时监控 top / htop

```
top
htop
```

核心查看内容：

- `us/sy/wa/id` CPU占用分布
- `load average` 1/5/15分钟负载
- 进程CPU占用排序

top常用快捷键：

- `P` CPU排序
- `M` 内存排序
- `1` 展开多核
- `z` 高亮
- `q` 退出

## 2. 查看CPU硬件信息（静态）

```
lscpu
cat /proc/cpuinfo
```

关键信息：

- 物理CPU数（Socket）
- 核心数 Core
- 逻辑核 CPU(s)（超线程）

## 3. 一次性获取系统负载（脚本可用）

```
uptime
```

# 二、内存资源查看

## 1. 查看整体内存、Swap（最常用）

```
free -h
```

重点只看两个：

- `available`：真实剩余可用内存
- `Swap used`：是否在用交换分区

## 2. 查看系统内存详细参数

```
cat /proc/meminfo
```

## 3. 查看进程内存占用

```
top
ps aux --sort=-%mem
```

内存字段释义：

- `VIRT` 虚拟内存（**无参考意义**）
- `RES` 物理内存（**真实占用**）
- `SHR` 共享内存

# 三、磁盘资源查看

## 1. 查看磁盘分区使用率

```
df -h
```

## 2. 查看目录/文件大小

```
du -sh *
du -sh /var/log
```

## 3. 查看磁盘IO状态

```
iotop
iostat 1
```

# 四、进程资源查看

## 1. 快照查看所有进程

```
ps aux    # 侧重资源占用 CPU/MEM
ps -ef    # 侧重父子进程 PPID
```

## 2. 进程树查看

```
pstree -p
```

## 3. 筛选进程

```
ps aux | grep python
pgrep -lf python
```

## 4. 进程状态认知（top/ps S字段）

- `R` 运行/排队
- `S` 休眠
- `D` 不可中断IO休眠
- `Z` 僵尸进程
- `T` 暂停

# 五、网络资源查看（重点补充）

## 1. 网卡信息、IP地址

```
ip a
ip addr
ip link
```

## 2. 路由信息

```
ip route
ip r
```

## 3. 端口、连接状态（生产必用）

```
# 查看所有监听端口
ss -tulnp

# 查看所有已建立连接
ss -tanp
```

状态：

- `LISTEN` 监听端口
- `ESTABLISHED` 正常连接
- `TIME-WAIT` 关闭等待

## 4. 网络流量查看

```
apt install iftop
iftop
```

## 5. 网络连通性测试

```
ping 网关/公网
curl 端口探测
telnet ip 端口
```

## 6. DNS解析测试

```
nslookup mirrors.tuna.tsinghua.edu.cn
```

# 六、系统整体状态快速巡检命令合集

```
uptime
free -h
df -h
ss -tulnp
ip a
top -b -n1
```

# 七、本篇总结（巡检标准）

1. CPU看：`top、lscpu、uptime`
2. 内存看：`free -h、top RES`
3. 磁盘看：`df、du、iostat`
4. 进程看：`ps、top、pstree`
5. 网络看：`ip a、ip r、ss、curl`

# 八、易错点（巡检专用）

1. 内存不要看 free，要看 available
2. 进程内存不要看 VIRT，要看 RES
3. 网络端口排查优先 ss，淘汰 netstat
4. load高不一定CPU忙，可能是IO阻塞

## 🔗 相关笔记

- [[Linux 性能排查：vmstat 与 iostat 完整运维笔记]]
- [[Shell04 系统监控巡检脚本学习笔记]]
- [[Linux常见故障排障方法]]
