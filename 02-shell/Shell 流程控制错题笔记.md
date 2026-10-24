---
title: Shell 流程控制错题笔记
created: 2026-10-15
updated: 2026-10-24
status: 待复习
tags: [shell, 脚本]
---

# Shell 流程控制考核 - 错题笔记

> 考核范围：if 参数校验 / if-elif 多分支 / for 循环 / while 循环 / case 选择 / 综合端口检查
> 日期：2026-10-15 · 环境：WSL Debian

---

## 错题 1｜return vs exit（p1 / p2 / p5）

**错误写法**：脚本顶层用 `return 1`

```
return: can only `return' from a function or sourced script
```

**原因**：`return` 只能用在**函数里**或 `source` 脚本时。独立运行脚本的顶层是"程序"，退出必须用 `exit`。

**✅ 正确**：
```bash
if [ $# -ne 1 ]; then
	echo "用法：$0 <名字>"
	exit 1
fi
```

> **口诀：`exit` 退出程序，`return` 退出函数。**

---

## 错题 2｜case 做数值比较（p2）

**错误写法**：
```bash
case $1 in
-ge 90)   # 永远匹配不上！
	echo "优秀"
```

**原因**：`case` 是**精确匹配/模式匹配**，不是数值比较。`-ge 90)` 被当成字面字符串 `"-ge 90"`，永远匹配不上，直接掉进 `*)`。数值比较（`-ge`）只能写在 `[ ]` / `[[ ]]` 里。

**✅ 正确**：
```bash
if [ $1 -ge 90 ]; then
	echo "优秀"
elif [ $1 -ge 60 ]; then
	echo "及格"
else
	echo "不及格"
fi
```

> **口诀：case 比"值"，if 比"条件"。** case 适合精确值（`start)`）、通配（`*`）、区间 glob（`9[0-9])`）；数值大小判断用 if/elif。

---

## 错题 3｜变量 `$` 丢失（p3）

**错误写法**：
```bash
txt="text{$i}.txt"   # {$i} 是字面文本，不是变量！
touch txt            # 缺 $！创建了名为 txt 的文件
```

**原因**：bash 变量引用必须带 `$`：
- `$i` = 取变量值
- `${i}` = 取变量值（边界清晰，推荐）
- `{i}` = 普通文本，bash 不认识

**✅ 正确**：
```bash
for ((i=1; i<=5; i++)); do
	txt="text${i}.txt"
	touch "$txt"
	[ -e "$txt" ] && echo "$txt 存在" || echo "$txt 创建失败"
done
```

> **口诀：引用变量永远写 `$i` 或 `${i}`。** 这是脚本跑不起来的最常见原因。

---

## 错题 4｜while 比较符 + 死循环（p4）

**错误写法**：
```bash
while [ "$i" <= 5 ]    # <= 非法
do
	echo "第{$i}次"    # 无递增 → 死循环
done
```

**原因**：
1. `[ ]` 里数值比较没有 `<=`，合法操作符是 `-le` → 报 `[: <=: unexpected operator`
2. 循环体里没有递增语句 → 条件永远成立 → **死循环**

**✅ 正确**：
```bash
i=1
while [ $i -le 5 ]
do
	echo "第 ${i} 次"
	i=$((i+1))        # 递增必须要有
done
echo "总共循环 $((i-1)) 次"
```

> **数值比较符全集**：`-eq`(等于) `-ne`(不等) `-gt`(大于) `-ge`(大于等于) `-lt`(小于) `-le`(小于等于)
> **while 循环必须有个"会变化的变量"，否则死循环。**

---

## 错题 5｜case 语法 + systemctl（p5）

**错误写法**：
```bash
case #1 in              # # 是注释符！行被截断 → 语法错误
start)
	systemctl -start nginx   # systemctl 子命令不带 -
```

**原因**：
1. `#` 在 bash 里是注释符，`#1` 让整行变成注释 → `case in` 语法炸
2. `systemctl` 的子命令是 `start/stop/restart/status`，**不带 `-`**

**✅ 正确**：
```bash
case "$1" in
start)
	service nginx start
	;;
stop)
	service nginx stop
	;;
restart)
	service nginx restart
	;;
status)
	service nginx status
	;;
*)
	echo "用法：$0 {start|stop|restart|status}"
	exit 1
	;;
esac
```

> **⚠️ 环境坑（本机）**：WSL 的 systemd 是 offline（pid1=init），`systemctl` 会卡死，**本机用 `service`**。普通 Linux 上 `systemctl start nginx` 才是标准。

---

## 错题 6｜判断对象错误 + 算术写法（p6）

**错误写法**：
```bash
if [ -n "$port" ];then      # port 永远非空 → 无条件 +1 → 永远"存活 3/3"
	num=$[ "$num" + 1 ]      # $[ ] 过时语法，且不能带引号
```

**原因**：
1. 要判断的是 **grep 结果** `$output` 是否非空（= 端口在监听），而不是循环变量 `$port`（永远有值）
2. `$[ ]` 是过时算术写法，里面不能有引号 → 应使用 `$(( ))`

**✅ 正确**：
```bash
num=0
for port in 9000 9001 9002
do
	output=$(ss -tlnp | grep ":$port ")
	if [ -n "$output" ]; then
		echo "$port 监听中"
		num=$((num+1))
	else
		echo "$port 未监听"
	fi
done
echo "存活 $num/3"
```

> **口诀：判断谁，测谁。** 算术统一用 `$(( ))`。

---

## 📊 四大系统性教训

| # | 错误规律 | 出错题 | 口诀 |
|---|---|---|---|
| 1 | `$` 丢失 / 花括号误用（`{$i}`、`touch txt`、`case #1`） | p3 p4 p5 | 引用变量永远写 `$i` 或 `${i}` |
| 2 | `return` 用在脚本顶层 | p1 p2 p5 | `exit` 退出程序，`return` 退出函数 |
| 3 | `[ ]` 比较符与 case 混用（`<=`、`-ge 90)`） | p2 p4 | 数值比较用 `if [ $x -le 5 ]`；case 只做精确匹配 |
| 4 | 判断对象错误 + 算术写法（`-n "$port"`、`$[ ]`） | p6 | 判断谁测谁；算术 `$((x+1))` |

---

## 🔧 附加实战坑：CRLF 换行符（Windows ↔ WSL）

**现象**：
```
$'\r': command not found
unexpected end of file / expecting "then"
```

**根因**：Windows 换行是 `\r\n`（回车+换行），Linux 是 `\n`（只有换行）。从 Windows 编辑的脚本每行末尾多一个 `\r`，shell 把 `\r` 当命令名。

**修复**：
```bash
# 单文件
sed -i 's/\r$//' p1.sh
# 批量
sed -i 's/\r$//' *.sh
# 专用工具
sudo apt install dos2unix && dos2unix p1.sh p2.sh
```

**预防**：
```bash
# git 自动转 LF
git config --global core.autocrlf input
```
- WSL 里用 vim/nano 写脚本 → 天然 LF，无此问题
- VS Code 写 → 右下角把 CRLF 切 LF
- 记事本写脚本 → 必踩此坑

---

## 📝 正确写法速查（最终版）

```bash
# if 参数校验
[ $# -ne 1 ] && { echo "用法：$0 <参数>"; exit 1; }

# if-elif 数值分级
[ $1 -ge 90 ] && echo 优秀 || { [ $1 -ge 60 ] && echo 及格 || echo 不及格; }

# for 批量创建
for ((i=1; i<=5; i++)); do touch "text${i}.txt"; done

# while 计数（先增后打印，避开死循环）
i=0; while [ $i -lt 5 ]; do i=$((i+1)); echo "第${i}次"; done

# case 服务控制（本机用 service）
case "$1" in start) service nginx start ;; *) echo 用法; exit 1 ;; esac

# 端口存活检查（判断 grep 结果）
output=$(ss -tlnp | grep ":$port ") && [ -n "$output" ] && echo 监听中
```

## 🔗 相关笔记

- [[Shell02 流程控制 复习笔记（bash 运维版）]]
