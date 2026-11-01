---
title: Linux资源查看命令速查（free-df-du-ss）
created: 2026-10-24
updated: 2026-10-24
status: 待复习
tags: [linux, 基础, 资源管理, 命令]
---

# Linux 资源查看命令速查（free / df / du / ss）

> 定位：配合 [[Linux03 Linux资源管理]] 的实战命令笔记，每个命令标注**核心作用 / 常用参数 / 高频易错 / 面试考点**。

## 一、free —— 查看内存使用

**核心作用**：一眼看清物理内存和 swap 的使用情况。

```bash
free -h    # 人类可读（G/M 单位），运维默认写法
```

输出关键列：
- `total` 总内存 / `used` 已用 / `free` 完全空闲
- **`available`（最重要）**：真正可分配给新进程的内存 = free + 可回收的缓存
- `buff/cache`：Linux 用空闲内存做缓存，属正常现象

**高频易错**：`free` 很小 ≠ 内存不足。看 `available`，别看 `free`。
**面试考点**：为什么 used + free ≠ total？→ 因为 buff/cache 被单独列出，缓存随用随释放。

## 二、df —— 查看磁盘空间（按分区）

```bash
df -h                # 各挂载点空间使用情况
df -hT               # 附带文件系统类型
df -i                # 查看 inode 使用率（磁盘没满但写不了文件的常见原因！）
```

**高频易错**：空间 100% 但删了大文件空间不释放 → 文件仍被进程持有，需重启该进程或 `lsof | grep deleted` 确认。
**面试考点**：`df` 按分区统计；inode 耗尽同样导致写入失败。

## 三、du —— 查看目录/文件占用（定位"谁"占空间）

```bash
du -sh *                    # 当前目录各项占用汇总（最常用的找大文件姿势）
du -h --max-depth=1 /var    # 只看 /var 下一层
du -ah / | sort -rh | head  # 全盘占用 Top 10（大磁盘慎用，慢）
```

**排查套路**：`df -h` 发现某分区满 → 进该分区逐层 `du -sh *` 下钻 → 锁定大目录/大文件。

## 四、ss —— 查看端口与连接（netstat 的现代替代）

```bash
ss -tlnp        # 查看监听中的 TCP 端口及所属进程（排查"服务起没起"首选）
ss -tlnp | grep 8080    # 检查指定端口
ss -s           # 连接状态统计（看 TIME_WAIT 是否堆积）
```

参数记忆：**t**cp + **l**isten + **n**umeric(不解析域名/服务名，快) + **p**rocess。

**面试考点**：`ss` 直接读内核数据结构，比 `netstat` 快得多；排查服务端口冲突用 `ss -tlnp`。

## 五、组合实战套路

```bash
# 内存异常排查
free -h && ps aux --sort=-%mem | head -5

# 磁盘写满排查
df -h → cd 异常分区 → du -sh * | sort -rh | head

# 端口不通排查
ss -tlnp | grep <端口>   # 没监听=服务没起；有监听=查防火墙
```

## 🔗 相关笔记

- [[Linux03 Linux资源管理]]
- [[Linux系统资源全方位巡检]]
- [[Linux 性能排查：vmstat 与 iostat 完整运维笔记]]
- [[Linux常见故障排障方法]]
- [[ps 命令完整讲解（运维+面试版）]]
