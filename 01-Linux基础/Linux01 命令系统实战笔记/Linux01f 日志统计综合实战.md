---
title: Linux01f 日志统计综合实战
created: 2026-10-16
updated: 2026-10-24
status: 待复习
tags: [linux, 基础]
---

> Nginx 日志统计 6 大任务 + 高频易错点速查 · 拆分自 [[Linux01 命令系统实战笔记]]

## 十二、综合实战：Nginx 日志统计

### 测试日志样例

```
192.168.1.12 - - [01/Oct/2026] "GET /d" 500
192.168.1.10 - - [01/Oct/2026] "GET /a" 200
192.168.1.10 - - [01/Oct/2026] "GET /b" 200
192.168.1.10 - - [01/Oct/2026] "POST /api/login" 200
192.168.1.15 - - [01/Oct/2026] "GET /test" 404
192.168.1.16 - - [01/Oct/2026] "GET /abc" 404
```

---

### 任务 1：统计每个 IP 访问次数，取 Top 3

**传统管道方案**：
```bash
tr -s ' ' access.log | cut -d' ' -f1 | sort | uniq -c | sort -nr | head -3
```

**awk 方案**：
```bash
awk '{ip[$1]++} END {for(k in ip) print ip[k], k}' access.log | sort -nr | head -3
```

---

### 任务 2：统计 HTTP 状态码分布

```bash
tr -s ' ' access.log | cut -d' ' -f9 | sort | uniq -c
# 或 awk
awk '{status[$9]++} END {for(k in status) print status[k], k}' access.log | sort -nr
```

---

### 任务 3：只提取 5xx 错误日志的 IP

```bash
grep ' 500$' access.log | awk '{print $1}'
# 或
awk '$9 >= 500 {print $1}' access.log
```

---

### 任务 4：统计 POST 接口访问量

```bash
grep POST access.log | awk '{print $1}' | sort | uniq -c
```

---

### 任务 5：sed + awk 综合清洗

需求：删除 404 行，替换请求方法，提取 IP/方法/状态码

```bash
sed '/ 404$/d' access.log \
| sed -E 's/"GET/METHOD_GET/; s/"POST/METHOD_POST/' \
| awk '{print $1, $6, $9}'
```

---

### 任务 6：sed 反向引用给 IP 加标签

```bash
sed -E 's/^([0-9]+\.[0-9]+\.[0-9]+\.[0-9]+)/client:\1/' access.log
```

---

## 十三、高频易错点速查表

| # | 易错点 | 正确认知 |
|---|---|---|
| 1 | `>` 会先清空文件 | 哪怕命令失败，文件也被清空 |
| 2 | `2>&1` 顺序 | 必须放在 `> file` 后面，不能反过来 |
| 3 | uniq 不用 sort | uniq 只合并相邻重复行，**必须先 sort** |
| 4 | find 找内容 | find 找文件，grep 找文件内容 |
| 5 | `&` 就是后台常驻 | `&` 只是当前终端后台，关终端就死；要 nohup |
| 6 | jobs 看所有进程 | jobs 只看当前终端的任务 |
| 7 | `kill -9` 随便用 | 先 `kill`(15) 优雅退出，不行再 `-9` |
| 8 | xargs 处理带空格文件名 | 用 `find -print0 \| xargs -0` |
| 9 | `-n1` 解决空格分割 | `-n1` 不改变分割规则，默认仍按空格切 |
| 10 | `*` 匹配任意字符 | `*` 是前面字符0次或多次；任意字符串用 `.*` |
| 11 | `.` 就是小数点 | `.` 匹配任意字符，小数点要写 `\.` |
| 12 | cut 处理多空格 | cut 只认单字符分隔，多空格用 awk |
| 13 | sed -i 直接改 | 先不加 -i 预览，确认后再加；推荐 `-i.bak` |
| 14 | 管道传 stderr | 管道只传 stdout，stderr 要 `2>&1 \|` 或 `\|&` |
| 15 | which 能查内置命令 | which 只查外部程序，内置命令用 type |
| 16 | tail -f 跟踪重建文件 | 文件重建用 `tail -F` |
| 17 | 目录没有 x 能进 | 目录必须有 x 才能 cd 进入 |
| 18 | nohup 后 jobs 能看到 | nohup 脱离作业管理，jobs 看不到，用 ps |

---

> 📌 **学习路线总结**：
> 文件查看(cat/less/head/tail) → 查找(find/which/type/man) → 权限(chmod/chown) → 进程(ps/top/kill/jobs/bg/fg/nohup) → 重定向与管道 → xargs → 文本工具(grep/cut/sort/uniq/tr/wc) → awk → sed → 正则 → 日志实战


## 🔗 相关笔记

- [[Linux01e 文本三剑客与正则]]
- [[nginx01 综合项目实战笔记]]
- [[Linux运维面试题整理]]
