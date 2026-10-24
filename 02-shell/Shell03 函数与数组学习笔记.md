---
title: Shell03 函数与数组学习笔记
created: 2026-10-15
updated: 2026-10-24
status: 待复习
tags: [shell, 脚本]
---

# Shell 函数与数组 学习笔记

> 定位：运维脚本骨架课 · 日期：2026-10-15
> 核心价值：数据区（数组）+ 处理区（函数）+ 主流程（for）三分离 = 可维护脚本

---

## 一、函数（Function）

### 核心语法
```bash
# 定义（先定义后调用，脚本从上往下执行）
say_hi() {
	echo "你好，$1"
}
# 调用
say_hi "运维"
```

### 4 条规则
| 规则 | 说明 |
|---|---|
| 先定义后调用 | 调用时函数必须已存在 |
| 函数内 `$1 $2` 是函数参数 | 会遮蔽脚本的 `$1 $2` |
| `return` 返回退出码 | 0-255；用 `$?` 取；**0=成功，非0=失败** |
| `local` 局部变量 | 只函数内有效，不污染全局 |

### 函数名直接当条件判断
```bash
is_ok() {
	[ -f /tmp/flag ] && return 0 || return 1
}
if is_ok; then   # if 判断"退出码 == 0 吗"
	echo "有 flag"
fi
```

---

## 二、数组（Array）

### 定义与操作
```bash
ports=(9000 9001 9002)
echo ${ports[0]}       # 9000（下标从 0 开始）
echo ${ports[@]}       # 全部元素
echo ${#ports[@]}      # 长度 3
ports[3]=9003          # 按下标加
ports+=(9004)          # 追加
unset ports[1]         # 删除下标 1
```

### 遍历（生产标准）
```bash
for p in "${ports[@]}"; do
	echo "端口 $p"
done
```

### 数组 vs 普通变量
数组 = 名单化：加一个端口只改数组一行，循环逻辑不用动。

---

## 三、实战：check_backends.sh（函数+数组版）

```bash
#!/bin/bash
check_port() {
	local port=$1  # 这里是函数的参数，不是脚本的
	if ss -tlnp | grep -q ":$port "; then
		echo "✅ $port 正在监听"
		return 0
	else
		echo "❌ $port 未被监听"
		return 1
	fi
}

ports=(9000 9001 9002)
ok=0
for p in "${ports[@]}"; do
	if check_port "$p"; then  # 函数的传参方式和脚本很像
		ok=$((ok + 1))
	fi
done
echo "存活 $ok/${#ports[@]}"
```

> 加端口只改 `ports=(...)` 一行；分母用 `${#ports[@]}` 自动适配。

---

## 四、本课 3 个坑（都是血泪）

### 坑 1：没看真实输出就写 grep
```bash
ss -tlnp | grep ":port ${port}"   # ❌ ss 输出里根本没有 "port" 字样
```
**教训：先看数据，再写匹配** —— 写 grep 前先 `ss -tlnp | head` 看真实格式。
ss 真实输出：`LISTEN 0 511 0.0.0.0:9000 ...`（冒号后直接是数字）
正确：`grep -q ":$port "`

### 坑 2：Shell 的 0/1 与 C 语言相反（跨语言经典误区）
| | 0 | 非 0 |
|---|---|---|
| C 语言 | 假 | 真 |
| Shell 退出码 | **成功（if=真）** | **失败（if=假）** |

原因：退出码本质是"错误码"——0=没有错误=成功；1/2/127=第几种错误。
**记忆：Shell 里 0 = 成功 = 真；非 0 = 失败 = 假。别拿 C 思维写 shell。**

### 坑 3：统计方向与标签不一致
```bash
num=3
成功时 num=$((num - 1))    # ❌ 输出的是"剩余数"，标签却写"存活"
echo "存活 $num"
```
**正确**：存活数 = 成功次数，从 0 开始加：
```bash
ok=0
成功时 ok=$((ok + 1))
echo "存活 $ok/${#ports[@]}"
```

---

## 五、写脚本三查 + 新口诀

```
三查：① 数括号 $(( )) 双括号成对
      ② 数美元：每个变量前都有 $，$ 后是变量名
      ③ 查空格：[[ 内容 ]] 首尾空格
新口诀：先看数据再写匹配；Shell 0=成功真，1=失败假
```

---

## 六、收官课：函数互相调用 + 数组当参数（分层架构）

### 分层设计：log → check_port → check_all

```bash
log() {                                    # 底层：基础能力（日志）
	echo "[$(date '+%H:%M:%S')] [$1] $2"
}
check_port() {                             # 中层：业务逻辑（检查端口）
	local port=$1
	log "run" "检查端口 $port"
	...
}
check_all() {                              # 上层：调度（遍历名单）
	for p in "$@"; do
		check_port "$p"
	done
}
```

**Why 分层**：`log` 只写一次。想给日志加颜色、改格式、写文件——**只改 log 一个函数**，所有调用点自动生效（解耦）。

### 数组当函数参数（传参技巧）

```bash
f arr          # ❌ 只传字符串 "arr"
f "${arr[@]}"  # ✅ 展开成多个独立参数 → 函数里 "$@" 逐个接收
```

### 毕业作品：monitor.sh（p8 优化版）

```bash
#!/bin/bash
# 三层架构 + 参数覆盖名单
log() {
	echo "[$(date '+%H:%M:%S')] [$1] $2"
}
check_port() {
	local port=$1
	if ss -tlnp | grep -q ":$port "; then   # 尾空格防误匹配 :90000
		log "run" "$port 正在监听"
		return 0
	else
		log "error" "$port 未监听"
		return 1
	fi
}
check_all() {
	local ok=0
	for p in "$@"; do
		check_port "$p" && ok=$((ok + 1))
	done
	echo "存活 $ok/$#"
}
ports=(9000 9001 9002 9999)
[[ $# -gt 0 ]] && ports=("$@")             # 有外部参数就用外部名单
check_all "${ports[@]}"
```

> 运行：`./monitor.sh`（默认名单）或 `./monitor.sh 9000 9001`（外部名单）
> ⚠️ 9001/9002 未监听 = 后端进程没启动（WSL 重启后需跑 start_backends.sh），不是脚本错误。

---

## 七、常用系统函数（内置命令）速查

| 命令 | 作用 | 示例 |
|---|---|---|
| `basename` | 取文件名（去路径） | `basename /etc/nginx/nginx.conf` → `nginx.conf` |
| `dirname` | 取目录部分 | `dirname /etc/nginx/nginx.conf` → `/etc/nginx` |
| `seq` | 生成数字序列 | `seq 1 5` → `1 2 3 4 5` |
| `read` | 读用户输入/文件行 | `read -r -p "确认?" ans` |
| `sleep` | 等待 N 秒 | `sleep 3` |
| `date` | 时间戳 | `date '+%F %T'` → `2026-10-15 21:27:10` |
| `$RANDOM` | 随机数 | `echo $((RANDOM % 100))` → 0-99 |
| `cut` | 按分隔符切列 | `echo "a:b:c" | cut -d: -f2` → `b` |
| `tr` | 字符转换 | `echo ABC | tr 'A-Z' 'a-z'` → `abc` |
| `mktemp` | 安全创建临时文件 | `tmp=$(mktemp)` |
| `wc` | 计数（行/字/字节） | `wc -l /var/log/nginx/access.log` |
| `sort` / `uniq` | 排序 / 去重计数 | `sort file | uniq -c` |

### 运维高频片段（直接抄）

```bash
# 取配置文件名（去路径去扩展名）
base=$(basename /etc/nginx/sites-available/mysite .conf)   # → mysite

# 读用户输入做确认（危险操作前必用）
read -r -p "确认删除? [y/N] " ans
[[ $ans == y || $ans == Y ]] || exit 0

# 等待服务起来（最多等 10 秒）
for i in $(seq 10); do
	curl -s -o /dev/null http://localhost:8080 && break
	sleep 1
done

# 随机端口/后缀（如 $RANDOM % 1000 + 8000 → 8000-8999）
echo $((RANDOM % 1000 + 8000))
```

## 🔗 相关笔记

- [[Shell04 系统监控巡检脚本学习笔记]]
- [[Shell02 流程控制 复习笔记（bash 运维版）]]
