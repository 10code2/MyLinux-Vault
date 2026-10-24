---
title: Linux01c 进程管理与信号
created: 2026-10-16
updated: 2026-10-24
status: 待复习
tags: [linux, 基础]
---

> ps/top/kill/后台任务/nohup + 面试高频信号 · 拆分自 [[Linux01 命令系统实战笔记]]

## 四、进程与后台任务：ps / top / kill / jobs / bg / fg / nohup

### 1. ps —— 查看进程快照（静态）

```bash
ps aux              # 最常用：列出所有进程（BSD 风格）
ps -ef              # 另一种风格（System V 风格）
ps aux | grep nginx # 过滤 nginx 相关进程
```

**`ps aux` 关键字段**：

| 字段      | 含义             |
| ------- | -------------- |
| USER    | 进程所属用户         |
| **PID** | 进程 ID（kill 靠它） |
| %CPU    | CPU 占用率        |
| %MEM    | 内存占用率          |
| VSZ     | 虚拟内存大小         |
| RSS     | 物理内存大小         |
| TTY     | 关联终端           |
| STAT    | 进程状态           |
| START   | 启动时间           |
| TIME    | 累计 CPU 时间      |
| COMMAND | 启动命令           |

**STAT 状态码**：
- R 运行（Running）
- S 可中断睡眠（Sleeping，最常见）
- D 不可中断睡眠（通常是 IO）
- T 被暂停（Stopped）
- Z 僵尸进程（Zombie）
- `<` 高优先级
- `N` 低优先级

[[ps 命令完整讲解（运维+面试版）]]

---

### 2. top —— 实时进程监控（动态）

```bash
top
```

**内部快捷键**：

| 按键 | 作用 |
|---|---|
| `P` | 按 CPU 占用排序 |
| `M` | 按内存占用排序 |
| `k` | 输入 PID 直接杀进程 |
| `1` | 展开显示每个 CPU 核心 |
| `q` | 退出 |
| `h` | 显示帮助 |

> 💡 `htop` 是 top 的升级版（需要安装），更直观，支持鼠标操作。

---

### 3. kill —— 发送信号给进程

```bash
kill 1234           # 默认发送 SIGTERM(15)，优雅终止
kill -15 1234       # 显式指定 SIGTERM
kill -9 1234        # SIGKILL，强制杀死
kill -1 1234        # SIGHUP，挂断（常用于重载配置）
killall nginx       # 按程序名杀所有 nginx 进程
pkill -f "python app.py"  # 按命令行匹配杀进程
```

🔑 **面试高频：SIGTERM vs SIGKILL**
- `kill -15` (SIGTERM)：默认信号，程序可以捕获、处理、保存数据后再退出，**优雅关闭**。
- `kill -9` (SIGKILL)：内核直接杀掉进程，程序无法捕获、无法清理，**暴力终止**。只在进程卡死无响应时才用。

---

### 4. jobs —— 查看当前终端的后台任务

```bash
jobs
jobs -l      # 额外显示 PID
```

输出示例：
```
[1]+  Stopped                 sleep 600
```
- `[1]`：**任务编号**（job number），不是 PID！`fg`/`bg` 用这个编号。
- `+`：最近的任务；`-`：次近的任务。

⚠️ **核心限制**：`jobs` **只能看当前终端的后台任务**！新开一个终端，jobs 列表是空的。跨终端看进程用 `ps`/`top`。

---

### 5. fg —— 后台任务切到前台

```bash
fg 1        # 将编号 1 的后台任务切到前台
fg          # 不带参数，默认操作最近的任务
```

前台程序占用终端，不能输入新命令。按 `Ctrl+C` 终止前台程序。

---

### 6. bg —— 暂停的任务在后台继续运行

```bash
bg 1        # 编号 1 的任务，后台继续运行
bg          # 默认最近任务
```

**典型操作流程**：
```bash
python app.py    # 前台运行
Ctrl+Z           # 暂停当前前台程序，放到后台（暂停状态）
jobs             # 查看任务编号
bg 1             # 在后台继续跑
fg 1             # 再切回前台
```

⚠️ **重要认知**：`bg` 只是在当前 shell 会话内部后台运行，**进程依然依附当前终端**！关闭终端照样被杀。bg ≠ nohup！

---

### 7. nohup —— 忽略挂断信号，脱离终端运行

```bash
# & 表示放到后台；nohup 让进程忽略 SIGHUP
nohup python app.py &
```

**输出默认重定向**：nohup 会把 stdout/stderr 自动写入当前目录的 `nohup.out`。

🔑 **面试必背：三种后台方式对比**

| 写法 | 后台运行 | 关闭终端后 | 能用 jobs 看到 |
|---|---|---|---|
| `cmd &` | ✅ | ❌ 被杀（SIGHUP） | ✅ |
| Ctrl+Z + bg | ✅ | ❌ 被杀（SIGHUP） | ✅ |
| `nohup cmd &` | ✅ | ✅ 继续运行 | ❌（脱离作业管理） |

⚠️ **常见坑**：
1. 只写 `nohup cmd` 不加 `&` → 前台卡住，没效果。
2. 只写 `cmd &` 不加 nohup → 关终端就死。
3. nohup 进程脱离了 shell 的作业管理，所以 `jobs` 看不到它，必须用 `ps` 查。

---

## 五、信号深入（面试高频）

### 四个容易混淆的信号

| 信号 | 编号 | 触发场景 | 默认动作 |
|---|---|---|---|
| **SIGHUP** | 1 | 关闭终端 / SSH 断开 | 终止进程 |
| **SIGINT** | 2 | 按 Ctrl+C | 终止进程 |
| **SIGQUIT** | 3 | 按 Ctrl+\ | 终止 + core dump |
| **SIGTSTP** | 20 | 按 Ctrl+Z | 暂停进程 |
| **SIGTTIN** | 21 | 后台任务读终端输入 | 暂停进程 |
| **SIGTTOU** | 22 | 后台任务写终端输出 | 暂停进程 |
| **SIGTERM** | 15 | kill 默认信号 | 优雅终止 |
| **SIGKILL** | 9 | kill -9 | 强制杀死（不可捕获） |

🔑 **SIGTTOU 详解（很多教材漏掉的点）**：

交互式 bash 开启作业控制时，**只有前台进程组拥有终端的读写权限**：
- 后台任务读终端（stdin）→ 收到 SIGTTIN → 进程暂停（Stopped）
- 后台任务写终端（stdout/stderr）→ 收到 SIGTTOU → 进程暂停（Stopped）

但是否触发取决于终端的 `tostop` 设置：
```bash
stty -a | grep tostop    # 查看当前设置
stty tostop              # 开启：后台写终端 → SIGTTOU → 暂停
stty -tostop             # 关闭：允许后台输出到终端（很多发行版默认）
```

> 部分环境/程序对 SIGTTOU 的处理是直接退出，表现为 jobs 显示 `Terminated`。标准默认信号动作是**暂停**，不是杀死。

---


## 🔗 相关笔记

- [[进程和线程]]
- [[ps 命令完整讲解（运维+面试版）]]
- [[Linux系统资源全方位巡检]]
