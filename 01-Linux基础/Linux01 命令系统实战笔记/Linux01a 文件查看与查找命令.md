---
title: Linux01a 文件查看与查找命令
created: 2026-10-16
updated: 2026-10-24
status: 待复习
tags: [linux, 基础]
---

> 文件查看（cat/less/head/tail/wc）+ 查找定位（find/which/type/man） · 拆分自 [[Linux01 命令系统实战笔记]]

## 一、文件查看与统计

### 1. cat —— 一次性输出文件全部内容（concatenate）

**核心作用**：读取文件并打印到终端，适合小文件。

```bash
cat test.txt                # 打印全部内容
cat -n test.txt             # -n：显示行号（空行也编号）
cat -b test.txt             # -b：只给非空行加行号
cat file1.txt file2.txt     # 拼接输出多个文件
cat > newfile.txt           # 新建文件，键盘输入，Ctrl+D 结束
```

⚠️ **易错点**：
- 大文件用 cat 会刷屏，无法分页浏览，大文件用 less。
- `cat > file` 会**先清空文件**，哪怕后面什么都没输入，文件也变成空的。

---

### 2. less —— 分页查看器（最推荐看大文件）

**核心作用**：分页浏览文件，按需加载，不会一次性读入全部内容，是 `more` 的增强版。

```bash
less log.txt
```

**内部快捷键（必背）**：

|按键|作用|
|:--|:--|
|`空格` / `f`|向下翻一页|
|`b`|向上翻一页|
|`↑` `↓`|逐行滚动|
|`/关键词`|向下搜索|
|`?关键词`|向上搜索|
|`n`|跳到下一个匹配|
|`N`|跳到上一个匹配|
|`g`|跳到文件开头|
|`G`|跳到文件末尾|
|`q`|退出 less|


✅ **实用技巧**：man 手册内部就是用 less 实现的，所以 less 的快捷键和 man 完全一样。

---

### 3. head —— 查看文件头部

**核心作用**：默认输出前 10 行。

```bash
head log.txt               # 默认前 10 行
head -n 5 log.txt          # 指定前 5 行
head -20 log.txt           # 简写，省略 n
head -c 20 log.txt         # -c：输出前 20 个字符
```

---

### 4. tail —— 查看文件末尾（日志神器）

**核心作用**：默认最后 10 行；最常用实时跟踪日志。

```bash
tail log.txt               # 最后 10 行
tail -n 15 log.txt         # 最后 15 行
tail -f log.txt            # -f：实时跟踪文件新增内容（最常用）
tail -F log.txt            # -F：文件被删除/重建也能继续跟踪（生产推荐）
tail -n +20 log.txt        # 从第 20 行开始输出到末尾
```

🔑 **面试高频：`-f` vs `-F` 的区别**
- `-f`：跟踪文件描述符，如果文件被删掉重建，tail 会继续跟踪旧的已删除文件，新文件不会被跟踪。
- `-F`：跟踪文件名，文件被删除重建后会自动重新打开新文件，生产环境推荐用 `-F`。

---

### 5. wc —— 统计行数、单词数、字符数（word count）

```bash
wc test.txt                # 输出：行数 单词数 字符数
wc -l test.txt             # -l：只统计行数（最常用）
wc -w test.txt             # -w：单词数量（空白分隔）
wc -c test.txt             # -c：字符数（字节）
wc -m test.txt             # -m：字符数（多字节字符按字符算）
```

**实用场景**：
- `wc -l /etc/passwd` 统计系统用户数
- `ls | wc -l` 统计当前目录文件数量

---

## 二、查找命令：find / which / type / man

### 1. find —— 递归遍历目录树查找文件

**核心作用**：按文件名、大小、时间、类型查找文件。
> ⚠️ **find 找的是文件本身，不是文件内容！** 找文件里的文字用 grep。

```bash
# 基础语法：find 搜索路径 条件
find . -name "*.txt"        # 当前目录递归找所有 .txt，区分大小写
find . -iname "*.TXT"       # -iname：忽略大小写
find /home -name "test.sh"  # 在 /home 下找名为 test.sh 的文件

# 按类型
find . -type f              # f=普通文件
find . -type d              # d=目录
find . -type l              # l=软链接

# 按大小
find . -size +10M           # 大于 10M
find . -size -100k          # 小于 100k
find . -size +1G            # 大于 1G

# 按时间
find . -mtime -3            # 3 天内修改过的文件
find . -mtime +30           # 30 天前修改过的文件
find . -atime -1            # 1 天内访问过的文件

# 组合动作
find . -name "*.log" -delete    # 找到直接删除（慎用！）
find . -name "*.txt" -exec ls -l {} \;  # 对每个结果执行 ls -l
```

🔑 **面试高频坑**：
1. 通配符 `*` 一定要加引号 `"*.txt"`，否则 shell 会先在当前目录展开，find 收到的不是通配符而是实际文件名。
2. `-delete` 非常危险，建议先 `find ... -print` 确认结果，再改成 `-delete`。
3. 文件名带空格时，`-exec` 和 `xargs` 都要注意，推荐用 `find -print0 | xargs -0`。

---

### 2. which —— 查找外部命令的可执行文件路径

**核心作用**：只在 `PATH` 环境变量里搜索二进制程序。

```bash
which ls
# 输出 /usr/bin/ls
which python3
```

⚠️ **特点**：
- ✅ 只找外部命令程序本体
- ❌ 找不到 shell 内置命令（如 `cd`、`pwd`），因为它们不是独立程序文件

---

### 3. type —— 判断命令类型（内置/外部/别名/函数）

```bash
type ls
# ls is aliased to `ls --color=auto'

type cd
# cd is a shell builtin

type python3
# python3 is /usr/bin/python3
```

🔑 **which vs type 对比（面试常问）**：

| 命令      | 能识别内置命令 | 能识别别名 | 能识别函数 |
| ------- | ------- | ----- | ----- |
| `which` | ❌       | ❌     | ❌     |
| `type`  | ✅       | ✅     | ✅     |


---

### 4. man —— 查看官方手册（manual）

```bash
man ls            # 查看 ls 手册
man find
man 1 ls          # 第1章：普通用户命令（默认）
man 2 open        # 第2章：系统调用（C 编程）
man 3 printf      # 第3章：C 库函数
```

man 内部操作和 less 完全一样（空格翻页、`/` 搜索、`q` 退出）。

> 💡 **实用技巧**：很多命令支持 `--help`，是精简版帮助，比 man 查得快。`man` 是最权威最完整的文档。

---


## 🔗 相关笔记

- [[which 和 type 命令的真实价值]]
- [[Linux02 零基础到文本处理总结]]
