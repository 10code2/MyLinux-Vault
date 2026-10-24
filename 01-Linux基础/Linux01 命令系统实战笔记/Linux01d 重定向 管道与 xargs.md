---
title: Linux01d 重定向 管道与 xargs
created: 2026-10-16
updated: 2026-10-24
status: 待复习
tags: [linux, 基础]
---

> 三大标准流 / 重定向全家桶 / 管道 / tee / xargs · 拆分自 [[Linux01 命令系统实战笔记]]

## 六、重定向与管道

### 三大标准流（必须记住）

| 名称 | 文件描述符 fd | 默认指向 |
|---|---|---|
| 标准输入 stdin | 0 | 键盘 |
| 标准输出 stdout | 1 | 终端屏幕 |
| 标准错误 stderr | 2 | 终端屏幕 |

重定向 = 修改这三个流的指向。

---

### 1. 输出重定向：`>` 和 `>>`

```bash
command > file       # stdout 覆盖写入 file（清空原有内容）
command >> file      # stdout 追加写入 file 末尾
```

⚠️ **核心：`>` 和 `>>` 默认只重定向 stdout！stderr 仍然打印屏幕。**

```bash
ls /notexist > out.log
# out.log 是空的（因为 ls 没有正常输出）
# 错误信息 "No such file or directory" 直接打印在屏幕上
```

---

### 2. 标准错误重定向：`2>` 和 `2>>`

```bash
ls /notexist 2> err.log      # stderr 覆盖写入 err.log
ls /notexist 2>> err.log     # stderr 追加写入 err.log
```

---

### 3. stdout + stderr 一起重定向

```bash
# 写法1（通用）：stdout 到文件，stderr 重定向到 stdout 的目标
ls /etc /notexist > all.log 2>&1

# 写法2（bash 特有简写）
ls /etc /notexist &> all.log
```

🔑 **面试必考：`2>&1` 顺序不能写反！**
```bash
# ❌ 错误：先 2>&1（此时 stdout 还指向屏幕），再 > all.log
ls /etc /notexist 2>&1 > all.log
# 结果：stdout 进了 all.log，但 stderr 还是打印屏幕

# ✅ 正确：先 > all.log（stdout 指向文件），再 2>&1（stderr 跟着 stdout 走）
ls /etc /notexist > all.log 2>&1
```

> 理解关键：`2>&1` 是"把 stderr 重定向到 stdout **当前**指向的地方"，不是"把 stderr 和 stdout 绑在一起"。所以顺序很重要。

---

### 4. 输入重定向：`<`

```bash
wc -l < test.txt       # 把 test.txt 内容作为 wc 的 stdin
```

很多命令支持直接传文件名（`wc -l test.txt`），`<` 更多用于脚本和 here-doc。

---

### 5. Here-doc：`<<` 多行输入

```bash
cat << EOF > msg.txt
第一行文字
第二行文字
第三行文字
EOF
```
- `EOF` 是自定义分界符（也可以写成其他词）
- 遇到单独一行的 `EOF` 就结束输入
- 常用于脚本里批量写配置文件

⚠️ **易错点**：重定向符号 `> msg.txt` 要放在 `cat` 后面，不能放在 EOF 后面！

---

### 6. 管道：`|`

```bash
命令1 | 命令2 | 命令3
```

**核心：把命令 1 的 stdout 作为命令 2 的 stdin。**

⚠️ **关键认知**：
1. 管道只传递 **stdout**，**不传递 stderr**！stderr 直接穿过管道打印屏幕。
2. 管道是内存数据流，不生成中间文件。
3. 管道每一段命令都在独立子 shell 中执行。

```bash
ps aux | grep ssh
cat /etc/passwd | wc -l
ls -l /etc | less
```

**`|&`（拓展）**：stdout + stderr 一起送入管道，等价于 `2>&1 |`
```bash
ls /etc /不存在 |& grep "No such"
```

---

### 7. tee —— 管道分支（一边写文件一边输出屏幕）

```bash
ls -l /etc | tee list.txt       # 结果存 list.txt，同时屏幕显示
ls /home | tee -a list.txt      # -a：追加写入
```

**典型场景**：运行一个长时间命令，既想看实时输出，又想存日志。

---

### 重定向符号速查表

| 符号 | 作用 |
|---|---|
| `>` | stdout 覆盖写入文件 |
| `>>` | stdout 追加写入文件 |
| `2>` | stderr 覆盖写入 |
| `2>>` | stderr 追加写入 |
| `&>` | stdout + stderr 覆盖写入 |
| `&>>` | stdout + stderr 追加写入 |
| `<` | 文件作为 stdin 输入 |
| `<<EOF` | here-doc 多行输入 |
| `\|` | 管道：前 stdout 作为后 stdin |
| `\|&` | stdout + stderr 送入管道 |

---

## 七、xargs（管道黄金搭档）

### 核心一句话

管道 `|` 把内容作为 **stdin** 传给后一个命令；
**xargs 把 stdin 里的文本拆分成【命令参数】。**

> 很多命令（`rm`、`cp`、`mv`、`mkdir`）**不支持从 stdin 读数据**，必须靠 xargs 转成参数。
> 而 `cat`、`grep`、`wc` 这类命令可以直接读 stdin，**不需要 xargs**。

### 基础示例

文件 `name.txt`：
```
file1.txt
file2.txt
file3.txt
```

```bash
cat name.txt | xargs touch
# 等价执行：touch file1.txt file2.txt file3.txt
```

为什么不能直接 `cat name.txt | touch`？因为 touch 不读 stdin！

---

### 常用参数

#### `-n 数字`：每次传几个参数

```bash
cat name.txt | xargs -n1 touch
# 等价：touch file1.txt
#       touch file2.txt
#       touch file3.txt
```

🔑 **面试考点：`-n1` vs 不带 `-n` 的区别**
- 不带 `-n`：**一次性传所有参数，只执行一次命令**（效率高，少创建进程）
- `-n1`：**每次只传 1 个参数，执行多次命令**（多次创建进程，开销大）
- ⚠️ `-n1` **不改变分割规则**！xargs 默认仍然按空格、Tab、换行分割。

#### `-I {}`：占位符（高频！）

`{}` 代表读到的内容，可以放在命令任意位置。
```bash
ls *.txt | xargs -I {} cp {} {}.bak
# 一次处理一行，等价：cp a.txt a.txt.bak
#                cp b.txt b.txt.bak
```

⚠️ 注意：`-I` 是大写 i！不是小写 L。`-I` 隐含了 `-n1`。

#### `-d`：指定分隔符

默认按空白（空格/Tab/换行）分割。
```bash
cat list.txt | xargs -d '\n' rm    # 只用换行做分隔
```

#### `-0`：配合 find 处理带空格文件名（生产最常用）

```bash
find . -name "*.log" -print0 | xargs -0 rm
```
- `find -print0`：输出文件名用 `\0`（空字符）分隔
- `xargs -0`：用 `\0` 分割
- **完美解决文件名包含空格、换行、特殊字符的问题**

#### `-r`：输入为空时不执行命令

```bash
find . -name "*.tmp" -print0 | xargs -0 -r rm
```
- 不加 `-r`：find 没找到文件 → xargs 执行裸 `rm` → 报 `rm: missing operand`
- 加 `-r`：输入为空就不执行命令
- ⚠️ `-r` 是 GNU xargs（Linux）特有，macOS/BSD 没有这个参数。

---


## 🔗 相关笔记

- [[Shell01 基础复习笔记]]
- [[Shell02 流程控制 复习笔记（bash 运维版）]]
