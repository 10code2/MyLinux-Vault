# 模拟面试 18 题复盘（CloudLab 项目实战版）

> 定位：基于 CloudLab 真实项目的一轮完整模拟面试，含开场话术、题目、标准答案、答题要点与个人薄弱点
> 日期：2026-10-22 · 形式：面试官提问 → 我作答 → 即时点评
> 配套：基础题见《Linux运维面试题整理》，运维通识见《运维基础概念》

---

## 〇、答题情况自评

**强项（答得有底气）**：
- 文件权限、软/硬链接
- 容器/数据卷、Shell 脚本、Prometheus
- CloudLab 项目讲解

**短板（需重点复习）**：
- HTTP 状态码分类（一开始记反了：3xx 是重定向、4xx 客户端、5xx 服务器）
- df/du、LVM、RAID
- 防火墙（iptables/firewalld）
- MySQL 索引/EXPLAIN

---

## 一、面试开场

### 1. 自我介绍（1 分钟，约 200 字）

> 面试官您好，我叫唐俊豪，2025 年毕业于佛山大学智能科学与技术专业，求职方向是 Linux 运维。我对运维很感兴趣，毕业后自己在一台云服务器上，用 Docker 从零搭建了一个多服务的自托管平台，包括网关、监控、密码管理、云盘这些，部署、配置和排错都是我独立完成的。之前我还在北京三维天地做过 ERP 实施实习，有驻场和客户沟通的经验。我学习能力比较强，也能踏实做事，希望能找一个运维岗位稳定下来，从基础做起，慢慢成长。

**要点**：读几遍改成自己的口气，能不看稿说出大意即可，不用逐字背。

### 2. CloudLab 项目讲解（3 分钟）

按这个顺序讲：

1. **开场**："毕业后在华为云买了一台 2核2G 的服务器，用 Docker 搭了 7 个服务，把运维常用的东西都实际跑了一遍。"
2. **架构**："最外面是 Nginx Proxy Manager，统一入口、子域名分流、负责 HTTPS；后面有 Vaultwarden、Uptime Kuma、Prometheus+Grafana、Alist、Halo，都用 Docker Compose 编排。"
3. **亮点**：
   - 证书：HTTP 验证失败，改 DNS 验证（阿里云 DNS 接口）成功签发；
   - 监控：两层（Prometheus+Node Exporter 主机指标，Uptime Kuma 探活），"对外走公网、对内走内网"；
   - 备份：Shell 脚本打包配置和数据卷，crontab 每天跑。
4. **踩坑**："内存只有 2G，Halo 是 Java 服务吃内存，限制 JVM 堆和容器内存、加 1G swap 才稳定；还处理了防火墙和 Docker 冲突、镜像加速。"
5. **收获**："完整走了部署、配置、监控、排错、备份全流程，证书验证、内存优化这些不自己踩一遍记不牢。"

---

## 二、容器与 Docker（第 1-4 题）

### 第 1 题：容器 vs 虚拟机

**本质区别（关键词）**：
- 虚拟机：虚拟化**硬件**，每台有独立的 Guest OS 和内核；
- 容器：**共享宿主机内核**，只隔离进程和运行环境。

| 维度 | 虚拟机 | 容器 |
|---|---|---|
| 内核/系统 | 独立 Guest OS | 共享宿主机内核 |
| 启动 | 分钟级 | 秒级（本质是进程） |
| 体量 | 几 GB~几十 GB | 几 MB~几百 MB |
| 隔离性 | 强 | 相对弱 |

**底层原理（加分）**：
- **Namespace（命名空间）**：负责隔离（PID、网络、文件系统等）；
- **Cgroups（控制组）**：限制 CPU/内存。

> 我的答题情况：资源开销、环境隔离、可复制性都答到了，但漏了"共享内核"这个最关键的词。

### 第 2 题：Docker 网络模式

| 模式 | 说明 |
|---|---|
| **bridge（默认）** | docker0 网桥，容器拿内网 IP，外部访问靠 `-p` 端口映射 |
| **host** | 直接用宿主机网络，不用 `-p` |
| **none** | 无网络，只有 lo |
| **container** | 与另一个容器共享网络 |

**Compose 网络（重点）**：
- docker-compose 启动多服务时，自动建一个自定义 bridge 网络，内置 DNS；
- 容器间可直接用"**服务名**"互访（如 NPM 填 `halo:8090`）。

**我的项目实际情况**：
- 每个服务独立 compose → 各自独立网络、互相隔离；
- 所以 NPM 无法直接用服务名，改用 `172.17.0.1`（docker0 网关）+ 端口映射绕路：

```
NPM 容器 → 172.17.0.1:8090（宿主机）→ 端口映射 → halo 容器
```

### 第 3 题：外部共享网络（怎么改）

**① 建外部网络**：

```bash
docker network create cloudlab
```

**② 每个 compose 声明加入**：

```yaml
services:
  halo:
    image: halohub/halo:latest
    networks:
      - cloudlab

networks:
  cloudlab:
    external: true
```

**③ NPM 配置**：
- Forward Hostname 填服务名/容器名（如 `halo`）；
- Forward Port 填**容器内部端口**（Halo 8090、Vaultwarden 80、Alist 5244，不是宿主映射端口）。

> 走容器名后，后端可不做 `-p` 映射，只给 NPM 保留 80/443。理解即可，真要改等入职后或有空再弄，别借"优化项目"占投递时间。

### 第 4 题：数据持久化（Volume vs Bind mount）

数据能放三个地方：

1. **容器可写层**：不挂载时，容器删除数据即失；
2. **Volume（数据卷）**：独立于容器，由 Docker 管理（`/var/lib/docker/volumes/`），容器删除卷还在；
3. **Bind mount（绑定挂载）**：挂宿主机指定目录（`-v /宿主路径:/容器路径`），数据是宿主机普通文件。

| | Volume | Bind mount |
|---|---|---|
| 位置 | Docker 管理 | 用户指定宿主机路径 |
| 管理 | `docker volume` | 普通目录操作 |
| 容器删除 | 卷保留 | 目录保留 |
| 适合 | 数据库、应用数据 | 配置文件、源码 |

> 我的项目：`monitor_grafana-data`、`monitor_prometheus-data` 是 named volume，独立于容器，所以能单独打包备份。
> 易错点：我一开始说"数据卷容器删除就消失"，说反了——消失的是容器可写层，不是数据卷。

---

## 三、排查与进程（第 5、11 题）

### 第 5 题：服务器卡顿 / 服务慢排查

**核心方法论：先整体后局部，先定位瓶颈层（CPU/内存/磁盘/网络）再深入，同时第一时间看日志。**

1. **整体负载**：
   - `uptime`：load average（1/5/15 分钟），对比 CPU 核数；
   - `top`：load、%Cpu(s)、内存，按 `P`/`M` 排序。
2. **CPU**：`%us`（用户态高=程序吃 CPU）、`%wa`（iowait 高=瓶颈在 IO）。
3. **内存**：
   - `free -h`：看 **available**（不是 free，buff/cache 可回收）；
   - `dmesg | grep -i oom`：看 OOM Killer。
4. **磁盘**：
   - `df -h`：空间是否满；`df -i`：inode 是否满（高频故障）；
   - `iostat -x 1`：%util、await。
5. **网络**：`ss -tlnp`（端口监听、连接数）；`ping`、`mtr`、`curl -I`。
6. **日志**：`journalctl -xe`、`/var/log/messages`、应用日志。

> 测试磁盘读写别轻易用 `dd` 写盘（生产环境风险），用 iostat 观察，专业压测用 `fio` 且在维护窗口。

### 第 11 题：进程管理

**查看进程**：
- `ps -ef`：标准格式，PPID 看父子关系；
- `ps aux`：%CPU、%MEM、STAT 状态；
- STAT：`R` 运行、`S` 可中断睡眠、`D` 不可中断睡眠（等 IO）、`Z` 僵尸、`T` 停止；
- `ps aux --sort=-%cpu | head`：按 CPU 排序。

**kill 信号**：
- `kill PID` 默认 **15（SIGTERM）**：优雅退出；
- `kill -9`（SIGKILL）：强制杀，不可捕获，可能丢数据，最后手段；
- `kill -1`（SIGHUP）：重载配置；
- 顺序：先 15 后 9。

**僵尸进程**：
- 子进程退出、父进程没回收（没 wait），停在 `Z` 状态；
- 不占 CPU/内存，但**占 PID 和进程表项**，大量会耗尽 PID；
- 不能直接 kill，要处理父进程（父死后被 systemd 收养回收）。

**后台运行**：
- `&`：后台但受终端控制；
- `nohup command > out.log 2>&1 &`：忽略 SIGHUP，关终端继续；
- Ctrl+Z 暂停，`jobs`/`fg`/`bg`；
- 更现代：**screen/tmux**；生产用 **systemd/Docker**。

---

## 四、网络（第 6、12、16 题）

### 第 6 题：输入 URL 到页面显示全过程

1. **DNS 解析**：
   - 浏览器缓存 → 系统缓存（hosts）→ 本地 DNS（递归）；
   - 本地 DNS 依次问：根 → 顶级域（.cloud）→ 权威 DNS（阿里云）；
   - 我的项目：cloud321.cloud 权威在阿里云，泛解析到服务器 IP。
2. **TCP 三次握手**。
3. **TLS 握手（HTTPS，重点）**：
   - 协商加密套件、服务器返回证书、客户端验证证书、交换会话密钥；
   - 之后 HTTP 加密；我项目里申请了证书、用 DNS 验证，必须能讲 TLS。
4. **发 HTTP 请求**：请求行 + `Host` 头（vault.cloud321.cloud）。
5. **服务端处理**：请求先到 NPM（网关，443），按 Host 子域名反代到对应容器。
6. **浏览器渲染**：HTML→DOM、CSS→CSSOM、渲染树→布局→绘制；断开（四次挥手）。

### 第 12 题：常见端口 + HTTP 状态码

**常见端口**：

| 端口 | 服务 |
|---|---|
| 22 | **SSH**（不是 SSL） |
| 80 / 443 | HTTP / HTTPS |
| 3306 | MySQL |
| 6379 | Redis |
| 8080 | HTTP 备用（不是数据库） |

> 补充：53 DNS、21 FTP、25 SMTP、110 POP3、143 IMAP。

**状态码分类（我一开始记反了）**：
- **2xx 成功；3xx 重定向；4xx 客户端错误；5xx 服务器错误。**

**常见码**：
- 200 成功；301 永久重定向；302 临时重定向；
- 400 请求错误；401 未认证；403 禁止；404 不存在（不是后端挂了）；
- 500 内部错误；
- 502 网关错误（后端挂了/端口没监听）；
- 503 服务不可用（过载/维护）；
- 504 网关超时（后端太慢）。

> 502/503/504 记忆：**502 后端挂了、503 忙不过来、504 后端太慢。**

### 第 16 题：TCP 握手/挥手 + 端口排查

**三次握手**：
1. 客户端 → 服务器：`SYN`（SYN_SENT）；
2. 服务器 → 客户端：`SYN+ACK`（SYN_RCVD）；
3. 客户端 → 服务器：`ACK`（ESTABLISHED）。

为什么三次：确认双方收发能力；两次无法确认客户端收到；防止失效历史连接。

**四次挥手**：
1. 主动方 → 被动方：`FIN`（FIN_WAIT_1）；
2. 被动方 → 主动方：`ACK`（CLOSE_WAIT，主动方 FIN_WAIT_2）；
3. 被动方发完 → 主动方：`FIN`（LAST_ACK）；
4. 主动方 → 被动方：`ACK`（TIME_WAIT，等 2MSL 关闭）。

为什么四次：TCP 全双工，两方向分别关闭；收到 FIN 时可能还有数据要发，先 ACK 后 FIN，不能合并。

> TIME_WAIT（2MSL）：保证最后 ACK 可重传；让旧连接滞留报文消散。

**端口连不上排查**：
1. `ss -tlnp` 看服务是否监听；
2. 监听地址：`127.0.0.1`（只本机）vs `0.0.0.0`（对外）；
3. 本机 `nc -zv 127.0.0.1 端口`；
4. 防火墙 iptables/firewalld；
5. 云安全组；
6. `ping`/`mtr`。

---

## 五、Nginx（第 7 题）

### 第 7 题：反向代理 / 负载均衡

**反向代理**：客户端请求发给代理，由代理转发给后端，客户端不知道真正后端。

**正向 vs 反向**：
- 正向代理：代理**客户端**（如 VPN），服务器不知道客户端；
- 反向代理：代理**服务端**（如 NPM），客户端不知道后端。

**作用**：统一入口、隐藏后端、SSL 终止（证书装在 NPM）、负载均衡、缓存压缩。

**负载均衡算法**：

| 算法 | 说明 |
|---|---|
| 轮询 round-robin | 默认，依次分配 |
| 加权轮询 weight | 性能好的多分 |
| ip_hash | 按客户端 IP 哈希，会话保持 |
| least_conn | 发给连接数最少的 |

底层是 Nginx 的 `upstream` 块。

> 关系：负载均衡是反向代理的功能之一；单后端只做转发，多后端才谈负载均衡。

---

## 六、Shell（第 9 题）

### 第 9 题：Shell 结构 + crontab

**脚本结构**：
- 开头 `#!/bin/bash`（shebang）；
- 代码体；
- 生产脚本开头加 `set -euo pipefail`。

**crontab 格式**：`分 时 日 月 周 命令`
- 我的备份：`30 3 * * * /opt/cloudlab/backup/backup.sh`；
- 常见：`*/5 * * * *`、`0 9 * * 1-5`。

**常见坑**：
1. **PATH 精简**（只有 /usr/bin:/bin）→ 命令用绝对路径或脚本开头 export PATH；
2. **任务重叠** → `flock -n /tmp/lock`；
3. **日志膨胀** → `>>` 追加 + logrotate；
4. **相对路径** → 用绝对路径或开头 cd；
5. **输出不处理会发邮件** → `>/dev/null 2>&1`。

---

## 七、监控（第 10 题）

### 第 10 题：Prometheus 原理

**Pull 模型**：Prometheus Server 主动按周期（scrape_interval）去各目标抓取指标。

**Exporter**：
- 跑在被监控机器上的小程序，采集指标并暴露 `/metrics`，**被动等 Prometheus 拉**；
- **不是 push 模型**；
- Node Exporter：9100 端口，采集主机指标。

**Pushgateway**：
- 短生命周期任务主动 push 给 Pushgateway，Prometheus 再拉；
- 这才是 push。

**数据链路**：
1. Node Exporter 暴露 9100/metrics；
2. Prometheus 按 prometheus.yml 的 scrape_configs 周期 pull；
3. 存入 **TSDB**（时序数据库），用 **PromQL** 查询；
4. Grafana 配 Prometheus 为数据源，渲染图表。

---

## 八、数据库（第 13 题）

### 第 13 题：MySQL

**索引**：
- 类似书的目录，底层主要是 **B+树**（O(log n)）；
- 没索引要全表扫描（O(n)）；
- 代价：占空间、拖慢写入，不是越多越好；
- 分类：主键、普通、唯一、联合索引。

**慢 SQL 排查**：
- **EXPLAIN（核心）**：
  - `type`：ALL 是全表扫描；
  - `key`：NULL 没走索引；
  - `rows`：扫描行数；
  - `Extra`：Using filesort/Using temporary 危险；
- 慢查询日志：`slow_query_log=ON`、`long_query_time=1`；
- `show full processlist`：看运行中的 SQL、锁等待；
- 索引失效：对索引列用函数、`like '%xx'` 左模糊、隐式类型转换、不满足最左前缀。

**连接命令**：
- `mysql -u root -p`（回车输密码）；远程加 `-h` `-P`；
- `show databases;`（有 s）；
- 先 `use 库名;` 才能 `show tables;`；
- `desc 表名;`；
- 建用户/授权：

```sql
create user 'user'@'localhost' identified by '密码';
grant all on db.* to 'user'@'localhost';
flush privileges;
```

---

## 九、文件与文本处理（第 14 题）

### 第 14 题：find/grep/awk/sed + 链接

**find**：`-name`、`-type f/d`、`-size +100M`、`-mtime -7`；`-exec rm {} \;`（慎用）。

**grep**：`-r`、`-i`、`-n`、`-v`、`-E`。

**awk**：`$1` 第一列、`$0` 整行、`NF` 列数、`NR` 行号；`-F:` 指定分隔符。

**sed**：`s/old/new/g` 替换；`-i` 直接改文件（先预览）；`-n '1,10p'` 打印行。

**软/硬链接**：
- 软链接（`ln -s`）：独立 inode，存路径，源文件删了失效，可跨文件系统/目录，像快捷方式；
- 硬链接（`ln`）：同 inode，链接数归零才删除，不跨文件系统；
- `ls -i` 看 inode。

---

## 十、服务与存储（第 15、17 题）

### 第 15 题：systemctl / systemd

**常用命令**：
- `start`/`stop`/`restart`/`reload`/`status`；
- `enable`/`disable`（开机自启）；
- `daemon-reload`；`list-units --type=service`。

**看日志**：`journalctl`
- `-u 服务名`；`-f` 实时；`--since`；`-xe`。

**systemd vs init**：
- CentOS 7 前（SysVinit，service）：**串行启动**，shell 脚本；
- CentOS 7 后（systemd）：**并行启动**，unit 文件，依赖管理，自动重启，集成 journald，cgroups；
- 现在 service 是兼容包装。

**unit 文件**（`/etc/systemd/system/xxx.service`）：

```ini
[Unit]
Description=My Service
After=network.target
[Service]
ExecStart=/opt/app/start.sh
Restart=always
[Install]
WantedBy=multi-user.target
```

改完 `systemctl daemon-reload && systemctl enable --now xxx`。

### 第 17 题：存储（df/du、LVM、RAID）

**df vs du**：
- `df`：整个文件系统容量（`-h`）；`df -i` 看 inode；
- `du`：目录/文件大小（`-sh`）。

**df 满但 du 没那么大**：
- 最经典：文件被删但进程还占用 → `lsof | grep deleted`，重启进程释放；
- 挂载点覆盖；inode 耗尽。

**LVM**：
- **PV（物理卷）→ VG（卷组）→ LV（逻辑卷）**；
- 好处：加盘进 VG，给 LV 在线扩容，不丢数据；支持快照。

**RAID**：

| 级别 | 最少盘 | 容量 | 特点 |
|---|---|---|---|
| RAID 0 | 2 | 全部 | 最快，一块坏全丢 |
| RAID 1 | 2 | 50% | 镜像，可坏一块 |
| RAID 5 | 3 | n-1 | 校验，可坏一块，常用 |
| RAID 10 | 4 | 50% | 又快又安全，贵 |

---

## 十一、防火墙（第 18 题）

### 第 18 题：iptables / firewalld

**iptables 四表**：
- **filter（过滤，默认）**、**nat（地址转换）**、mangle（改标记）、raw（豁免追踪）。

**五链**：PREROUTING、INPUT、FORWARD、OUTPUT、POSTROUTING。

**实用命令**：
- 查看：`iptables -L -n --line-numbers`；
- 放行：`iptables -I INPUT -p tcp --dport 8080 -j ACCEPT`；
- 动作：ACCEPT/DROP/REJECT；
- 删除：`iptables -D INPUT 序号`；
- `iptables-save` 保存。

**firewalld**：
- `firewall-cmd --list-all`；
- `--add-port=8080/tcp`（`--permanent` 永久，`--reload`）；
- `--add-service=http`。

**Docker 与防火墙（重点）**：
- Docker 靠 iptables 实现网络：
  - `-p` 端口映射靠 nat 表 **DNAT**；
  - 容器上网靠 **MASQUERADE（SNAT）**；
  - 容器间转发靠 FORWARD 和 DOCKER 链；
- 所以 Docker 一启动，iptables 多出 DOCKER 链；
- 冲突：手动 `iptables -F` 或 firewalld 可能误删 Docker 规则；
- 正确：**自定义规则加到 `DOCKER-USER` 链**（Docker 不清空，最先执行）。

---

## 复习清单（按优先级）

**必补短板**：
1. HTTP 状态码分类 + 502/503/504；
2. df/du、LVM、RAID；
3. iptables/firewalld、DOCKER-USER；
4. MySQL EXPLAIN、索引。

**保持强项**：权限、链接、容器/数据卷、Shell、Prometheus、项目讲解。

**原则**：会用、能举例，不死背理论；重心回到投递，面试前过一遍，临场自然答得出来。
