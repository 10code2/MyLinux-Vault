---
title: GitHub笔记-本地同步与换行符处理
created: 2026-10-21
updated: 2026-10-24
status: 待复习
tags: [排障, 复盘]
---

# GitHub 笔记：本地仓库同步 + 换行符（LF/CRLF）处理

> 适用环境：Windows Git Bash（MINGW64）+ WSL2 双环境管理仓库（如 Obsidian 库）
> 更新日期：2026-10-21

---

## Part 1｜本地仓库同步到 GitHub

### 1.1 What：同步的本质

Git 是分布式版本控制。仓库有三层状态：

```
工作区（你改的文件）→ 暂存区（git add 后的快照）→ 本地仓库（git commit 提交）→ 远程仓库（GitHub，git push 推送）
```

「同步到 GitHub」= 把本地已提交的 commit **推送（push）** 到远程仓库。**本地没 commit 的东西，永远推不上去。**

### 1.2 How：核心操作（日常更新三步曲）

```bash
# 1. 查看当前改动状态（养成习惯，先看再动）
git status

# 2. 把改动加入暂存区
git add .                    # 添加全部改动
# 或 git add 具体文件：git add host_monitor.sh

# 3. 提交到本地仓库（写清 commit message，别写 "update"）
git commit -m "feat: 新增磁盘巡检脚本，阈值超过80%输出告警"

# 4. 推送到 GitHub
git push
```

> 核心口诀：**add → commit → push**，三步缺一不可。

### 1.3 When：三种场景

**场景 A：本地已有项目，首次推送到 GitHub（新建空仓库）**

```bash
cd 你的项目目录

# 1. 关联远程仓库（https 或 ssh 方式，二选一）
git remote add origin https://github.com/10code2/ops-host-monitor.git
# 或 SSH：git remote add origin git@github.com:10code2/ops-host-monitor.git

# 2. 首次推送，-u 记住关联关系，以后直接 git push 即可
git push -u origin main
```

非 git 仓库先初始化：

```bash
git init
git add .
git commit -m "init: 项目初始化"
```

**场景 B：日常更新** → 就是上面的三步曲。

**场景 C：本地和远程都有更新（如网页端改过 README）**

```bash
# 推荐 rebase 方式，历史更干净
git pull --rebase origin main

# 冲突解决后：
git add .
git rebase --continue

# 最后推送
git push
```

### 1.4 Why：原理（理解原理才能应对报错）

1. **为什么 push 前要先 commit？** 远程只接收「已提交的快照」，工作区的修改不算数，commit 是推送的最小单位。
2. **为什么会被 rejected？** 远程仓库有你本地没有的提交，Git 拒绝覆盖历史，必须先合并（pull --rebase）。
3. **为什么 2021 年后密码登录 GitHub 失效？** GitHub 已禁用账号密码推送，必须用 **Personal Access Token（PAT）** 或 SSH key。

### 1.5 常见报错对照表

| 报错信息 | 原因 | 解决方案 |
|---|---|---|
| `! [rejected] ... (fetch first)` | 远程有本地没有的提交 | `git pull --rebase origin main` 后重新 push |
| `Authentication failed` / 提示输入密码失败 | 用了账号密码，GitHub 已禁用 | 生成 PAT（Settings→Developer settings→Personal access tokens），作为密码输入 |
| `Permission denied (publickey)` | SSH key 没配好 | `ssh-keygen -t ed25519` 生成，把公钥加到 GitHub→Settings→SSH keys |
| `remote origin already exists` | 已关联过远程 | `git remote -v` 查看，改错用 `git remote set-url origin 新地址` |
| `error: RPC failed; HTTP 413` / 文件过大 | 单文件超过 100MB 限制 | 用 `git lfs`，或把大文件加入 `.gitignore` |
| `fatal: not a git repository` | 没在仓库目录内执行 | `cd` 到仓库目录，或先 `git init` |

### 1.6 个人最佳实践

```bash
# 首次配置身份
git config --global user.name "10code2"
git config --global user.email "你的邮箱"

# WSL2/Linux 侧换行符策略（与 Windows 侧配合）
git config --global core.autocrlf input
```

- `.gitignore` 必配：`*.pro.user`、`*.db`、`.env`、`report/` 等不该入库
- commit message 规范：`feat:` / `fix:` / `docs:` / `chore:` 开头

---

## Part 2｜换行符（LF / CRLF）处理 ★新增

### 2.1 警告解读：这是什么

```text
warning: in the working copy of 'xxx.md', LF will be replaced by CRLF the next time Git touches it
```

翻译：**工作区文件当前是 LF 结尾（`\n`，Linux 风格）。Git 检查到 `core.autocrlf` 配置，会在下一次 checkout 时把文件里的 LF 全部替换成 CRLF（`\r\n`，Windows 风格）。**

这只是**预告，不是错误**。git add 照常成功，不影响操作。

### 2.2 原理剖析：三种换行符 + autocrlf

| 类型 | 符号 | 使用方 |
|---|---|---|
| LF | `\n` | Linux / macOS / Git 仓库内部标准 |
| CRLF | `\r\n` | Windows 记事本、旧版工具 |
| CR | `\r` | 上古 Mac，基本绝迹 |

**`core.autocrlf` 三值对比：**

| 值 | commit 进仓库时 | checkout 到工作区时 | 典型场景 |
|---|---|---|---|
| `true` | CRLF → LF | LF → CRLF | **Windows 默认**，仓库 LF、本地 CRLF |
| `input` | CRLF → LF | 不转换 | Linux / macOS 默认 |
| `false` | 不转换 | 不转换 | 跨平台混乱，不推荐 |

警告出现的原因：Windows 下 autocrlf 大概率是 `true`，而 `.md` 文件是 LF 结尾（在 WSL2/Linux 编辑过，或工具生成时保持 LF），Git 预告下次 checkout 会转成 CRLF。

### 2.3 When：什么时候必须处理

| 情况 | 处理 |
|---|---|
| 只有自己用，且只在 Windows 编辑 | 可忽略，警告无害 |
| 仓库同时被 Windows + WSL2 使用（常见） | **必须统一**，否则两边来回改会产生大量假 diff |
| 仓库里有 `.sh` 脚本，将来拷到 Linux 服务器跑 | **必须统一为 LF**，CRLF 会让脚本报 `\r: command not found` |
| 团队协作、跨平台多人开发 | **必须用 .gitattributes**，这是唯一正确姿势 |

> ⚠️ 运维真实事故：Windows 写的脚本传到 Linux 一跑就报 `\r: command not found`，根因就是 CRLF。运维岗高频考点。

### 2.4 How：解决方案（三档）

**方案 B（推荐，生产环境最佳实践）：仓库级统一 —— `.gitattributes`**

在仓库根目录创建 `.gitattributes`：

```text
# 所有文本文件按内容自动判断
* text=auto

# Markdown 和 shell 脚本强制 LF（避免脚本在 Linux 上报错）
*.md  text eol=lf
*.sh  text eol=lf
*.py  text eol=lf
```

执行一次让仓库按新规则规范化：

```bash
git add --renormalize .
git commit -m "chore: 统一仓库换行符为 LF"
```

> 为什么是最佳实践：`.gitattributes` 是**写进仓库的声明**，跟着仓库走，任何人在任何平台 clone 都生效；`core.autocrlf` 是**每台机器各自的本机配置**，换机器就失效。团队/跨平台场景必须用前者。

**方案 C（个人快速处理）：只改本机配置**

```bash
# 仓库内统一为 LF（commit 时转 LF，checkout 不转回 CRLF）
git config core.autocrlf input
```

仅对本仓库生效，不跟随仓库走。

**方案 A（不推荐长期用）：直接忽略警告**，仅限自己用、只在 Windows 编辑的场景。

### 2.5 双环境（Windows Obsidian + WSL2）实操建议

1. 在 `D:/Obsidian/Linux` 仓库根目录创建 `.gitattributes`（用 2.4 方案 B 模板）；
2. 执行 `git add --renormalize .` 再 commit 一次，警告消失；
3. WSL2 侧 `git config core.autocrlf input`，两边策略对齐；
4. 以后写 `.sh` 脚本，在 WSL2 里写完直接提交，永远保持 LF，避免部署到服务器踩 `\r` 的坑。

---

## 面试考点速记

- Git 推送流程：add → commit → push（远程只接收已提交快照）
- GitHub 已禁用密码登录，必须 PAT 或 SSH key
- 换行符：Git 仓库内部标准是 LF；Windows 工作区常见 CRLF
- `.gitattributes` 是仓库级换行符声明的唯一正确方式（跨平台团队协作）
- CRLF 会让 Linux 上的 `.sh` 脚本报 `\r: command not found`（运维高频事故）

## 🔗 相关笔记

- [[which 和 type 命令的真实价值]]
