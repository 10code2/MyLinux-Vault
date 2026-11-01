---
title: ps aux 实操案例笔记一
created: 2026-11-01
status: 进行中
tags:
  - 命令
  - 实践
---


> 环境：华为云 ECS hcss-ecs-035c 两条命令：
> 
> 1. `ps aux --sort=-%cpu | head -6` 按CPU从高到低排序
> 2. `ps aux --sort=-rss | head -6` 按物理内存RSS从高到低排序 表头字段统一说明： USER 运行用户 | PID 进程号 | %CPU CPU占比 | %MEM 内存占比 | VSZ 虚拟内存(KB) | RSS 常驻物理内存(KB) | TTY 终端 | STAT 状态 | START 启动时间 | TIME 累计CPU时间 | COMMAND 进程命令

---

## 案例1：按CPU排序 `ps aux --sort=-%cpu | head -6`

原始输出：

```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root     24925  0.4  8.1 32897008 153972 ?     Ssl  10月31   6:13 uptime-kuma
65534    13246  0.2  2.7 1854704 50900 ?       Ssl  10月19  54:25 /bin/prometheus --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/prometheus --storage.tsdb.retention.time=15d --web.enable-lifecycle
root     21881  0.2  0.3 157884  6476 ?        Ss   14:05   0:01 sshd: root@pts/2,pts/3
root        36  0.1  0.0      0     0 ?        S    10月18  37:40 [kswapd0]
65534    13254  0.1  0.7 1277568 14768 ?       Ssl  10月19  30:34 /bin/node_exporter --path.procfs=/host/proc --path.sysfs=/host/sys --path.rootfs=/rootfs --collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($|/)
```

逐行解释：

1. `root 24925 0.4 8.1 32897008 153972 ? Ssl 10月31 6:13 uptime-kuma`
    - 进程：uptime-kuma（监控告警面板）
    - 用户名root，PID=24925
    - %CPU=0.4%：当前CPU占用是0.4%，是这一批里CPU最高的
    - RSS=153972KB ≈150MB，占整机内存8.1%
    - TTY=`?`：后台守护进程，没有交互式终端
    - START：10月31日启动；TIME=6:13：**这个进程一共消耗了6分13秒CPU时间**，不是它跑了6分13秒
2. `65534 13246 0.2 2.7 1854704 50900 ? Ssl 10月19 54:25 /bin/prometheus ...`
    - 65534 是nobody用户，容器内非root运行prometheus
    - CPU 0.2%；TIME 54:25，这个进程虽然十几天前就启动了，但大部分时间休眠，累计CPU只有54分钟
    - 参数含义：保留指标15天、开启热重载
3. `root 21881 0.2 0.3 157884 6476 ? Ss 14:05 0:01 sshd: root@pts/2,pts/3`
    - sshd进程：你当前有2个ssh连接（pts/2、pts/3），root登录
    - 14:05才启动，占用极低
4. `root 36 0.1 0.0 0 0 ? S 10月18 37:40 [kswapd0]`
    - 带`[]`：内核线程，kswapd0负责内存swap交换回收
    - 它的TIME很高，代表长期在做内存换页；如果%CPU持续走高，说明内存不够
5. `65534 13254 0.1 0.7 1277568 14768 ? Ssl 10月19 30:34 /bin/node_exporter ...`
    - node_exporter，采集服务器指标给prometheus
    - 挂载宿主机/proc /sys，过滤虚拟文件系统，避免采集异常

👉 这条命令结论（实例）：**当前所有进程CPU都很低，没有CPU负载高的程序**

---

## 案例2：按内存RSS排序 `ps aux --sort=-rss | head -6`

原始输出：

```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root     31561  0.1 30.2 2897172 568908 ?      Ssl  10月24  16:58 java -Xmx256m -Xms256m -XX:SharedArchiveFile=application.jsa -jar application.jar
root     24925  0.4  8.1 32897008 153972 ?     Ssl  10月31   6:14 uptime-kuma
root     20487  0.0  5.2 11868772 99068 ?      Sl   10月19   1:58 node --abort_on_uncaught_exception --max_old_space_size=250 index.js
472      29679  0.0  3.1 1572028 58936 ?       Ssl  10月31   1:30 grafana server --homepath=/usr/share/grafana --config=/etc/grafana/grafana.ini --packaging=docker cfg:default.log.mode=console cfg:default.paths.data=/var/lib/grafana cfg:default.paths.logs=/var/log/grafana cfg:default.paths.plugins=/var/lib/grafana/plugins cfg:default.paths.provisioning=/etc/grafana/provisioning
65534    13246  0.2  2.7 1854704 50900 ?       Ssl  10月19  54:26 /bin/prometheus --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/prometheus --web.enable-lifecycle
```

逐行解释：

1. `root 31561 0.1 30.2 2897172 568908 ? Ssl 10月24 16:58 java -Xmx256m -Xms256m -XX:SharedArchiveFile=application.jsa -jar application.jar`
    - Java业务程序，PID=31561
    - RSS=568908KB ≈556MB，占整机内存30.2%，**本机内存第一大户**
    - JVM参数 `-Xmx256m`：JVM堆最大256M
    - ✅ 实例重点：堆上限只给256M，但进程RSS达到556M，因为RSS包含堆、元空间、线程栈、JSA归档、JVM本身开销，不是堆溢出
2. `root 24925 0.4 8.1 32897008 153972 ? Ssl 10月31 6:14 uptime-kuma`
    - 和上面案例是同一个uptime-kuma，内存稳定150MB左右
3. `root 20487 0.0 5.2 11868772 99068 ? Sl 10月19 1:58 node --abort_on_uncaught_exception --max_old_space_size=250 index.js`
    - Node服务，限制老生代内存250M，RSS≈96MB
4. `472 29679 0.0 3.1 1572028 58936 ? Ssl 10月31 1:30 grafana server ...`
    - Grafana可视化面板，UID=472是grafana默认用户，RSS≈57MB
5. `65534 13246 0.2 2.7 1854704 50900 ? Ssl 10月19 54:26 /bin/prometheus ...`
    - 和案例1的prometheus是同一个进程，RSS≈49MB

👉 这条命令结论（实例）：**CPU压力很小，内存瓶颈在这个Java应用**

---

## 两条命令对比（结合本次实例）

1. `ps aux --sort=-%cpu | head -6`：按CPU排序，适合**找CPU跑高的进程**。本次实例里最高才0.4%，无CPU异常
2. `ps aux --sort=-rss | head -6`：按RSS物理内存排序，适合**找吃内存的进程**。本次实例一眼看到Java程序占了30.2%内存

## 配套实操命令（针对本次实例里的PID）

```
# 看整机内存swap
free -h
# 持续观察内存变化，看java RSS会不会持续上涨（排查内存泄漏）
watch -n2 "ps aux --sort=-rss | head -6"
# 看这个java进程GC状态
jstat -gc 31561
# 查看进程运行了多久
ps -p 31561 -o pid,etime,cmd
```

