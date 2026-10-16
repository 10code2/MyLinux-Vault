# Shell04 系统监控巡检脚本学习笔记

> 定位：生产级巡检脚本实战 · 日期：2026-10-16
> 产出：`/usr/local/bin/monitor.sh` + crontab 每天 8:00 自动执行
> 前置：Shell03 函数与数组（本笔记大量复用函数分层 + 数组名单）

---

## 一、生产巡检脚本的 5 条设计原则

| 原则 | 为什么 | 怎么做 |
|---|---|---|
| 配置与代码分离 | 运维改阈值/名单，不改逻辑 | 顶部"配置区"放数组和阈值 |
| 统一记录格式 | 输出能被日志系统/人一眼看懂 | `record()` 函数统一打 PASS/WARN/ERROR |
| 退出码规范 | 监控系统（Zabbix/Prometheus）靠退出码判断 | `0=正常 1=告警 2=严重错误` |
| 日志落盘 | 出问题要回溯 | 同时输出屏幕 + 写 `/var/log/monitor.log` |
| 只读无副作用 | 巡检不能改系统状态 | 所有检查都是读命令（df/free/ss/ps） |

### 架构图

```
┌─────────────────────────────────────┐
│  配置区（阈值 + 检查名单数组）         │  ← 运维只改这里
├─────────────────────────────────────┤
│  底层函数：log() / record()          │  ← 日志 + 统一记录
├─────────────────────────────────────┤
│  业务函数：check_cpu / check_mem ... │  ← 每个维度一个函数
├─────────────────────────────────────┤
│  主流程 main()：依次调用 + 汇总 + 退出码 │
└─────────────────────────────────────┘
```

---

## 二、完整脚本（最终版，含所有修复）

```bash
#!/bin/bash
# ============================================================
# monitor.sh —— 生产级系统巡检脚本
# 用法：bash monitor.sh
# 退出码：0=全部正常  1=有告警  2=有严重错误
# ============================================================

# ==================== 配置区（运维只改这里） ====================
CPU_LOAD_WARN=2.0          # 1分钟 load 告警阈值（一般设为 CPU 核数）
MEM_FREE_WARN=10           # 内存可用率低于多少 % 告警
DISK_USE_WARN=80           # 磁盘使用率高于多少 % 告警
INODE_USE_WARN=80          # inode 使用率高于多少 % 告警
SWAP_USE_WARN=50           # swap 使用率高于多少 % 告警

CHECK_PORTS=(8080 8443 9000 9001 9002)   # 必须监听的端口
CHECK_PROCS=(nginx python3)              # 必须运行的进程
CHECK_DISKS=(/ /mnt/c /mnt/d)            # 必须检查的挂载点

LOG_FILE="/var/log/monitor.log"          # 日志落盘路径

# ==================== 全局状态 ====================
PASS_COUNT=0
WARN_COUNT=0
ERROR_COUNT=0

# ==================== 底层：日志 ====================
log() {
	local level=$1 msg=$2 ts
	ts=$(date '+%Y-%m-%d %H:%M:%S')
	echo "[$ts] [$level] $msg"
	echo "[$ts] [$level] $msg" >> "$LOG_FILE" 2>/dev/null
}

# 统一记录检查结果（所有 check_* 都调它）
record() {
	local status=$1 item=$2 detail=$3
	case $status in
		PASS)  PASS_COUNT=$((PASS_COUNT+1));  log "INFO"  "✅ $item — $detail" ;;
		WARN)  WARN_COUNT=$((WARN_COUNT+1));  log "WARN"  "⚠️  $item — $detail" ;;
		ERROR) ERROR_COUNT=$((ERROR_COUNT+1)); log "ERROR" "❌ $item — $detail" ;;
	esac
}

# ==================== 检查函数 ====================

# 1. CPU 负载
check_cpu() {
	local load over
	load=$(uptime | awk -F'load average:' '{print $2}' | awk -F',' '{print $1}' | tr -d ' ')
	over=$(awk -v l="$load" -v t="$CPU_LOAD_WARN" 'BEGIN{print (l>t)?1:0}')
	if [ "$over" = "1" ]; then
		record "WARN" "CPU负载" "1分钟load=$load > 阈值$CPU_LOAD_WARN"
	else
		record "PASS" "CPU负载" "1分钟load=$load（阈值$CPU_LOAD_WARN）"
	fi
}

# 2. 内存（注意：awk 匹配 /^Mem:/ 大小写敏感！）
check_mem() {
	local total available free_pct low
	total=$(free -m | awk '/^Mem:/{print $2}')
	available=$(free -m | awk '/^Mem:/{print $7}')
	free_pct=$(awk -v a="$available" -v t="$total" 'BEGIN{printf "%.1f", a/t*100}')
	low=$(awk -v f="$free_pct" -v t="$MEM_FREE_WARN" 'BEGIN{print (f<t)?1:0}')
	if [ "$low" = "1" ]; then
		record "WARN" "内存" "可用率=${free_pct}% < 阈值${MEM_FREE_WARN}%（${available}M/${total}M）"
	else
		record "PASS" "内存" "可用率=${free_pct}%（${available}M/${total}M）"
	fi
}

# 3. 磁盘使用率
check_disk() {
	for mp in "${CHECK_DISKS[@]}"; do
		local use
		use=$(df -P "$mp" 2>/dev/null | awk 'NR==2{print $5}' | tr -d '%')
		if [ -z "$use" ]; then
			record "ERROR" "磁盘$mp" "挂载点不存在或无法读取"
			continue
		fi
		if [ "$use" -ge "$DISK_USE_WARN" ]; then
			record "WARN" "磁盘$mp" "使用率=${use}% >= 阈值${DISK_USE_WARN}%"
		else
			record "PASS" "磁盘$mp" "使用率=${use}%"
		fi
	done
}

# 4. inode（含健壮性：WSL 的 DrvFs 不支持 inode，显示 "-"）
check_inode() {
	for mp in "${CHECK_DISKS[@]}"; do
		local use
		use=$(df -iP "$mp" 2>/dev/null | awk 'NR==2{print $5}' | tr -d '%')
		if [[ ! "$use" =~ ^[0-9]+$ ]]; then
			record "PASS" "Inode$mp" "不支持 inode 统计（跳过）"
			continue
		fi
		if [ "$use" -ge "$INODE_USE_WARN" ]; then
			record "WARN" "Inode$mp" "使用率=${use}% >= 阈值${INODE_USE_WARN}%"
		else
			record "PASS" "Inode$mp" "使用率=${use}%"
		fi
	done
}

# 5. 端口监听
check_ports() {
	for port in "${CHECK_PORTS[@]}"; do
		if ss -tlnp 2>/dev/null | grep -q ":$port "; then
			record "PASS" "端口$port" "监听中"
		else
			record "ERROR" "端口$port" "未监听"
		fi
	done
}

# 6. 关键进程
check_procs() {
	for proc in "${CHECK_PROCS[@]}"; do
		if pgrep -x "$proc" >/dev/null 2>&1; then
			record "PASS" "进程$proc" "运行中"
		else
			record "ERROR" "进程$proc" "未运行"
		fi
	done
}

# 7. 内核错误日志
check_dmesg() {
	local err_count
	err_count=$(dmesg --level=err,crit,alert,emerg 2>/dev/null | wc -l)
	if [ "$err_count" -gt 0 ]; then
		record "WARN" "内核日志" "存在 ${err_count} 条 error/crit 级日志"
	else
		record "PASS" "内核日志" "无 error 级日志"
	fi
}

# 8. swap（虚拟内存）
check_swap() {
	local total used used_pct over
	total=$(free -m | awk '/^Swap:/{print $2}')
	used=$(free -m | awk '/^Swap:/{print $3}')
	[ "$total" = "0" ] || [ -z "$total" ] && { record "PASS" "虚拟内存" "未启用 swap"; return; }
	used_pct=$(awk -v a="$used" -v t="$total" 'BEGIN{printf "%.1f", a/t*100}')
	over=$(awk -v u="$used_pct" -v t="$SWAP_USE_WARN" 'BEGIN{print (u>t)?1:0}')
	if [ "$over" = "1" ]; then
		record "WARN" "虚拟内存" "使用率=${used_pct}% > 阈值${SWAP_USE_WARN}%（${used}M/${total}M）"
	else
		record "PASS" "虚拟内存" "使用率=${used_pct}%（${used}M/${total}M）"
	fi
}

# ==================== 主流程 ====================
main() {
	log "INFO" "===== 巡检开始 ====="
	check_cpu
	check_mem
	check_disk
	check_inode
	check_ports
	check_procs
	check_dmesg
	check_swap

	local total=$((PASS_COUNT + WARN_COUNT + ERROR_COUNT))
	echo ""
	log "INFO" "===== 巡检汇总 ====="
	log "INFO" "总检查项: $total | 通过: $PASS_COUNT | 告警: $WARN_COUNT | 严重: $ERROR_COUNT"

	if [ "$ERROR_COUNT" -gt 0 ]; then
		log "ERROR" "巡检结果: 存在严重错误"
		exit 2
	elif [ "$WARN_COUNT" -gt 0 ]; then
		log "WARN" "巡检结果: 存在告警"
		exit 1
	else
		log "INFO" "巡检结果: 全部正常"
		exit 0
	fi
}

main "$@"
```

---

## 三、10 个命令详解

| 命令 | 作用 | 关键参数/用法 |
|---|---|---|
| `uptime` | 开机时间 + 1/5/15 分钟 load | load < 核数=正常，>核数=CPU排队 |
| `free -m` | 内存使用（MB） | 看第 7 列 `available`（真正可用），不是 `free` |
| `df -P` | 磁盘使用率 | `-P`=POSIX 格式，强制一行不换行；第 5 列=使用率 |
| `tr -d '%'` | 删除指定字符 | 把 "80%" 变 "80"（纯数字才能比较） |
| `df -iP` | inode 使用率 | `-i`=查索引节点；生产必查（小文件耗尽 inode 的坑） |
| `grep -q` | 安静匹配 | 不输出内容，只返退出码；脚本里"有没有"判断一律用 -q |
| `pgrep -x` | 精确查进程名 | 不会匹配到 grep 自己（ps\|grep 的经典坑）；-x=完全相等 |
| `dmesg --level=err` | 内核错误日志 | 硬件故障/OOM/磁盘错误先出现在这里 |
| `awk -v a="$x"` | 传 shell 变量进 awk | awk 有独立变量空间，必须用 -v 传；BEGIN 块做纯计算 |
| `inode`（概念） | 文件索引节点 | 存元数据，文件名存在目录项；总数固定，满了写不进文件 |

### awk 提取数值（运维最常用技巧）

```bash
free -m | awk '/^Mem:/{print $2}'      # /^Mem:/ 只处理以 Mem: 开头的行，取第 2 列
df -P / | awk 'NR==2{print $5}'        # NR==2 只处理第 2 行（跳过表头）
uptime | awk -F'load average:' '{print $2}' | awk -F',' '{print $1}'
```

### awk -v 浮点比较（bash 只能比整数）

```bash
over=$(awk -v l="$load" -v t="$CPU_LOAD_WARN" 'BEGIN{print (l>t)?1:0}')
# -v 传变量；BEGIN 不读文件直接执行；三目运算输出 1/0
[ "$over" = "1" ] && echo "超阈值"
```

---

## 四、踩坑记录（血泪）

| 坑 | 现象 | 原因 | 修复 |
|---|---|---|---|
| **inode DrvFs** | `integer expression expected`，使用率=`-%` | WSL 的 /mnt/c /mnt/d 是 DrvFs，不支持 inode，df -i 输出 `-` | 加 `[[ ! "$use" =~ ^[0-9]+$ ]]` 判断，非数字跳过 |
| **awk 大小写** | 内存/swap 显示 `-nan%（M/M）` | 脚本里写 `/^MEM:/`，free 输出是 `Mem:`（首字母大写其余小写），awk 正则大小写敏感 | 改成 `/^Mem:/` 和 `/^Swap:/` |
| **除零 nan** | `0/0*100 = nan` | 变量为空时 awk 算 0/0 | 加空值判断（swap total=0 时跳过） |
| **cron 未安装** | `crontab: command not found` | Debian 最小镜像不含 cron | `apt install -y cron` |
| **vim Esc 失效** | 按 Esc 显示 `^[` | 终端不识别 ANSI 转义 | 用 `Ctrl+[` 代替 Esc，或 `EDITOR=nano crontab -e` |
| **crontab 覆盖** | 写入新任务后旧任务消失 | `echo \| crontab -` 是整体替换，不是追加 | 追加用 `(crontab -l; echo '新任务') \| crontab -` |

---

## 五、crontab 定时部署

### 格式

```
分  时  日  月  周  命令
0   8   *   *   *   /usr/local/bin/monitor.sh >> /var/log/monitor_cron.log 2>&1
```

### 覆盖 vs 追加

```bash
# 覆盖（全部替换）
echo '任务' | crontab -

# 追加（保留旧任务）
(crontab -l 2>/dev/null; echo '新任务') | crontab -

# 查看
crontab -l
```

### 生产三坑

| 坑 | 解决 |
|---|---|
| cron 的 PATH 很精简（只有 /usr/bin:/bin） | 脚本开头显式设 `PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin` |
| 任务重叠（上一个没跑完下一个又启动） | 用 flock：`/usr/bin/flock -n /tmp/monitor.lock 脚本` |
| 日志越来越大 | 用 logrotate 轮转（daily + rotate 7 + compress） |

### WSL 注意

- WSL 不是 7x24 服务器，关机/休眠时 cron 不跑（配置方法和真实服务器一致）
- `service cron start` 确保服务运行
- 验证：把时间改成 1 分钟后，等触发看 `/var/log/monitor_cron.log`

---

## 六、毕业成果

- ✅ 一个生产级巡检脚本（8 个检查维度，可扩展）
- ✅ 配置与代码分离（改阈值不改逻辑）
- ✅ 退出码规范（0/1/2，可接监控系统）
- ✅ crontab 每天 8:00 自动执行 + 日志落盘
- ✅ 健壮性处理（WSL DrvFs、空值、除零）
- ✅ 10 个运维高频命令实战掌握

**扩展方向**：加邮件/钉钉报警（退出码非 0 时发通知）、加日志关键字检查（grep ERROR）、加服务自动恢复（检测到端口挂了自动 service start）。
