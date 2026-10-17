# Linux 命令系统实战笔记（零基础 → 面试可背版）

> 本笔记覆盖：文件查看、查找、权限、进程管理、管道重定向、xargs、文本三剑客（grep/awk/sed）、正则、日志统计实战。
> 每个命令都标注：**核心作用 / 常用参数 / 高频易错 / 面试考点 / 实用技巧**。

---

## 目录

- [一、文件查看与统计](#一文件查看与统计)
- [二、查找命令：find / which / type / man](#二查找命令find--which--type--man)
- [三、文件权限：chmod / chown](#三文件权限chmod--chown)
- [四、进程与后台任务：ps / top / kill / jobs / bg / fg / nohup](#四进程与后台任务ps--top--kill--jobs--bg--fg--nohup)
- [五、信号深入（面试高频）](#五信号深入面试高频)
- [六、重定向与管道](#六重定向与管道)
- [七、xargs（管道黄金搭档）](#七xargs管道黄金搭档)
- [八、文本处理基础工具](#八文本处理基础工具)
- [九、awk 详解](#九awk-详解)
- [十、sed 详解](#十sed-详解)
- [十一、Shell 正则表达式](#十一shell-正则表达式)
- [十二、综合实战：Nginx 日志统计](#十二综合实战nginx-日志统计)
- [十三、高频易错点速查表](#十三高频易错点速查表)

---

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

## 三、文件权限：chmod / chown

### 权限基础（必须先看懂）

`ls -l` 输出示例：
```
-rwxr-xr--  1 user group  1234 Oct  1 10:00 test.sh
```

拆解 `-rwxr-xr--`：
1. **第 1 位**：文件类型
   - `-` 普通文件
   - `d` 目录
   - `l` 软链接
   - `b` 块设备
   - `c` 字符设备
2. **第 2~4 位**：属主（user）权限
3. **第 5~7 位**：属组（group）权限
4. **第 8~10 位**：其他（other）权限

**权限字母与数字**：

| 字母 | 含义 | 数字 |
|---|---|---|
| r | read 读 | 4 |
| w | write 写 | 2 |
| x | execute 执行 | 1 |

> 数字权限 = r + w + x 相加：
> - rwx = 4+2+1 = 7
> - rw- = 4+2+0 = 6
> - r-x = 4+0+1 = 5
> - r-- = 4+0+0 = 4

示例：`rwxr-xr--` → 属主 7、属组 5、其他 4 → 数字 `754`

---

### 1. chmod —— 修改文件/目录权限（change mode）

**两种写法：数字法（最常用）、符号法**

#### 数字写法
```bash
chmod 755 test.sh        # u:rwx g:r-x o:r-x（可执行脚本常用）
chmod 644 file.txt       # u:rw- g:r-- o:r--（普通文件默认）
chmod 600 .ssh/id_rsa    # 私钥必须 600！只有属主可读
chmod -R 700 mydir/      # -R：递归，目录及内部所有文件一并修改
```

#### 符号写法（u/g/o/a）
```bash
chmod u+x test.sh        # 给属主增加执行权限
chmod g-w file.txt       # 去掉组的写权限
chmod o+r file.txt       # 给其他人增加读权限
chmod a+x run.sh         # a=all，所有人加执行权限
chmod u-x,g+w file       # 多个操作用逗号分隔
```

⚠️ **高频考点：目录权限的特殊性**
- 目录必须有 `x` 权限才能 `cd` 进入！
- 目录有 `r` 但没有 `x`，只能 `ls` 看到文件名，无法进入目录、无法访问文件内容。
- 普通文件默认 644，可执行脚本默认 755。

---

### 2. chown —— 修改属主/属组（change owner）

```bash
chown user1 test.txt              # 只改属主为 user1
chown user1:group1 test.txt       # 同时改属主和属组
chown :group1 test.txt            # 只改属组
chown -R user1:group1 myfolder/   # -R 递归
```

⚠️ **注意**：修改别人文件的所有者需要 root / sudo 权限。

---

## 四、进程与后台任务：ps / top / kill / jobs / bg / fg / nohup

### 1. ps —— 查看进程快照（静态）

```bash
ps aux              # 最常用：列出所有进程（BSD 风格）
ps -ef              # 另一种风格（System V 风格）
ps aux | grep nginx # 过滤 nginx 相关进程
```

**`ps aux` 关键字段**：

| 字段 | 含义 |
|---|---|
| USER | 进程所属用户 |
| **PID** | 进程 ID（kill 靠它） |
| %CPU | CPU 占用率 |
| %MEM | 内存占用率 |
| VSZ | 虚拟内存大小 |
| RSS | 物理内存大小 |
| TTY | 关联终端 |
| STAT | 进程状态 |
| START | 启动时间 |
| TIME | 累计 CPU 时间 |
| COMMAND | 启动命令 |

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

## 八、文本处理基础工具

### 1. grep —— 按关键词过滤行

```bash
grep "404" access.log       # 找包含 404 的行
grep -v "200" access.log    # -v：反向，取不匹配的行
grep -i "get" access.log    # -i：忽略大小写
grep -n "500" access.log    # -n：显示行号
grep -c "200" access.log    # -c：只输出匹配行数
grep -r "error" ./dir/      # -r：递归目录下所有文件
grep -E "GET|POST" access.log  # -E：扩展正则
```

> grep 是**行过滤**，整行保留，不切割字段。

---

### 2. cut —— 按分隔符截取列

```bash
cut -d' ' -f1 access.log     # -d 指定分隔符，-f 取第几列
cut -d' ' -f7 access.log     # 取第 7 列
cut -d' ' -f1,3 access.log   # 取第 1 和第 3 列
cut -d' ' -f1-3 access.log   # 取第 1 到第 3 列
cut -c1-10 file.txt          # -c：按字符位置截取
```

⚠️ **坑**：`cut` 只能用**单个固定字符**做分隔符。多个连续空格会被当成多个分隔，导致空字段。日志空格多时优先用 awk。

---

### 3. sort —— 排序

```bash
cat ip.txt | sort            # 默认按字典序排序
sort -n num.txt              # -n：按数字大小排序
sort -nr num.txt             # -nr：数字倒序（从大到小）
sort -u file.txt             # -u：去重（排序+去重一步到位）
sort -t',' -k2 -n data.csv   # -t 指定分隔符，-k 按第2列排序
```

🔑 **必背：uniq 必须配合 sort 使用！**
`uniq` 只消除**相邻**重复行。不排序直接 uniq，统计结果错误。

---

### 4. uniq —— 相邻行去重 + 计数

```bash
sort access.log | uniq           # 去重
sort access.log | uniq -c        # -c：计数，输出「次数 内容」
sort access.log | uniq -d        # -d：只输出重复行，出现超过一次
sort access.log | uniq -u        # -u：只输出不重复行，只出现一次
```

**输出格式**：`  128 10.0.0.5`（前面是次数，后面是内容）

---

### 5. tr —— 字符替换/删除/压缩（translate）

tr 只做**单个字符**的转换，不能处理单词，不能用正则。

```bash
echo abc | tr 'a-z' 'A-Z'        # 小写转大写
echo "a1b2c3" | tr -d '0-9'     # -d：删除所有数字
echo "a  b   c" | tr -s ' '     # -s：压缩连续空格为1个
echo "hello" | tr 'lL' 'xX'     # 字符替换映射
```

🔑 **`-s` 是日志预处理神器**：连续多个空格压缩成一个，再给 cut 用。

---

### 6. wc —— 统计

```bash
wc -l access.log        # 行数（最常用）
wc -w access.log        # 单词数
wc -c access.log        # 字节数
```

---

## 九、awk 详解

### 核心认知

awk = **逐行扫描文本，按分隔符切分字段，执行逻辑处理**。
默认以**空白字符**（连续空格/Tab）为分隔符，**自动处理连续空格，不需要 tr -s**。

```
$1  $2  $3  ...  $n
$0 = 整行
```

### 基础语法

```bash
awk '条件 {动作}' 文件
```
- 没有条件：所有行都执行动作
- 没有动作：默认 `{print}` 打印整行

```bash
# 打印所有 IP（第1列）
awk '{print $1}' access.log

# 打印状态码（第7列）
awk '{print $7}' access.log

# 过滤 500 状态码，打印整行
awk '$7 == 500' access.log

# 过滤 500，只打印 IP
awk '$7 == 500 {print $1}' access.log
```

⚠️ **`==` 是比较，`=` 是赋值，千万别写混！**

---

### 内置变量

| 变量 | 含义 |
|---|---|
| `$0` | 整行文本 |
| `$1..$n` | 第 n 个字段 |
| **`NF`** | Number of Fields：当前行总列数 |
| **`NR`** | Number of Record：当前行号（从1开始） |
| `FS` | 输入字段分隔符（默认空白） |
| `OFS` | 输出字段分隔符（默认空格） |

```bash
awk '{print NR, NF, $1, $7}' access.log
# 输出：行号 列数 IP 状态码
```

---

### 用 awk 做统计（替代 sort|uniq）

```bash
# 统计状态码分布
awk '{stat[$7]++} END {for(k in stat) print stat[k], k}' access.log
```

拆解：
1. `stat[$7]++`：数组 `stat`，key 是状态码，每读一行对应 key 计数 +1
2. `END {}`：**全部读完后才执行**
3. `for(k in stat)`：遍历数组

降序排列：
```bash
awk '{stat[$7]++} END {for(k in stat) print stat[k], k}' access.log | sort -nr
```

**统计 Top IP**：
```bash
awk '{ip[$1]++} END {for(k in ip) print ip[k], k}' access.log | sort -nr | head -3
```

---

### BEGIN 和 END 块

```bash
awk 'BEGIN {print "count status"} {stat[$7]++} END {for(k in stat) print stat[k],k}' access.log
```
- `BEGIN {}`：读文件**之前**执行（初始化、打表头）
- `END {}`：读文件**之后**执行（输出统计结果）

---

### `-F` 自定义分隔符

```bash
awk -F',' '{print $1, $3}' data.csv       # 逗号分隔
awk -F'||' '{print $1, $3}' data.log      # 多字符分隔
awk -F'[ -]' '{print $1, $2}' data.txt    # 正则：空格或横杠
awk 'BEGIN{FS=","} {print $1}' data.csv    # 等价写法
```

**经典：Nginx 日志带双引号**
```bash
# 以双引号分隔，第2段就是完整请求
awk -F'"' '{print $2}' access.log
# 输出：GET /api/user HTTP/1.1
```

---

### 内置函数

| 函数 | 作用 |
|---|---|
| `split(字符串, 数组, 分隔符)` | 字符串切割到数组 |
| `length($0)` | 当前行字符长度 |
| `sub(/旧/, 新, $0)` | 替换**第一个**匹配 |
| `gsub(/旧/, 新, $0)` | **全局**替换所有匹配 |
| `substr(字符串, 开始, 长度)` | 截取子串 |
| `tolower()` / `toupper()` | 大小写转换 |

```bash
# 把行里所有 192.168 替换成 X.X
awk '{gsub("192.168", "X.X"); print $0}' access.log

# split 示例
awk -F'"' '{split($2, req, " "); print req[1], req[2]}' access.log
# req[1]=请求方法 GET, req[2]=请求路径 /api/user
```

---

### 条件判断

```bash
awk '{
  if ($7 == 200) print $1, "OK"
  else if ($7 >= 400) print $1, "ERROR"
  else print $1, "OTHER"
}' access.log
```

---

### awk vs cut/grep 对比

| 功能 | 传统方案 | awk 方案 |
|---|---|---|
| 提取 IP | `tr -s ' ' \| cut -d' ' -f1` | `awk '{print $1}'` |
| 过滤 500 行 | `grep " 500 "` | `awk '$7==500'` |
| 统计状态码 | `tr\|cut\|sort\|uniq\|sort` | awk 数组直接计数 |

**awk 优势**：自动处理连续空格、内置数组做计数、支持条件和计算。

---

## 十、sed 详解

### 核心认知

sed = stream editor，**流式编辑器，逐行处理文本**。
默认只输出到屏幕，**不修改原文件**（加 `-i` 才改）。

```bash
sed [选项] '地址+动作' 文件
```

---

### 核心选项

| 选项 | 作用 |
|---|---|
| `-n` | 静默模式，只打印匹配的行（配合 `p`） |
| `-i` | **直接修改原文件**（高危！推荐 `-i.bak` 自动备份） |
| `-e` | 多个编辑表达式 |
| `-E` | 开启扩展正则（不用转义 `() + |`） |

⚠️ 不加 `-i`，所有修改只在屏幕预览，源文件不变。**先预览确认，再加 -i。**

---

### 常用动作指令

#### `p` 打印（配合 `-n`）

```bash
sed -n '2p' access.log          # 只打印第 2 行
sed -n '2,5p' access.log        # 打印第 2~5 行
sed -n '/500/p' access.log      # 打印包含 500 的行
```

#### `d` 删除（不加 -i 不改原文件）

```bash
sed '1d' test.txt               # 删除第 1 行
sed '/^$/d' test.txt            # 删除空行（^$ 空行正则）
sed '/404/d' access.log         # 删除包含 404 的行
```

#### `s` 替换（最常用）

格式：`s/旧/新/修饰符`

```bash
sed 's/200/OK/' access.log       # 每行第一个 200 替换成 OK
sed 's/200/OK/g' access.log      # g=global，一行内全部替换
sed -n 's/200/OK/gp' access.log # -n + p：只打印被替换的行
sed -i.bak 's/404/NotFound/g' access.log  # 直接改原文件，备份 .bak
```

**修饰符**：
- `g`：全局替换（不加只换每行第一个）
- `i`：忽略大小写
- `p`：打印替换成功的行（配合 `-n`）

**分隔符可以换**：内容有 `/`（URL）时换成 `#` 或 `|`
```bash
sed 's#/api#/v2/api#g' access.log
```

**反向引用 `\1 \2`**（`-E` 开启）：
```bash
sed -E 's/([0-9]+\.[0-9]+\.[0-9]+\.[0-9]+)/IP:\1/g' access.log
```

---

### 地址匹配两种方式

1. **数字地址**：
   - `3` 第 3 行
   - `3,7` 第 3~7 行
   - `3,$` 第 3 行到末尾（`$` 最后一行）
   - `$` 最后一行

2. **正则地址**：`/正则/`
```bash
sed '/500/s/GET/POST/g' access.log    # 只在包含 500 的行替换
```

---

### 多表达式

```bash
sed -e '/^$/d' -e 's/200/OK/g' access.log
# 或用分号
sed '/^$/d; s/200/OK/g' access.log
```

---

### sed vs awk 区分（面试常问）

| 工具 | 擅长 | 场景 |
|---|---|---|
| **sed** | 行操作、替换、删除、增加行 | 修改文件内容 |
| **awk** | 列提取、数值统计、逻辑判断 | 分析、提取数据 |

> 经典组合：`grep` 过滤 → `sed` 清洗替换 → `awk` 提取字段 → `sort/uniq` 统计

---

## 十一、Shell 正则表达式

### 基础元字符（BRE 基础正则）

| 符号 | 含义 | 示例 |
|---|---|---|
| `.` | 任意单个字符（不含换行） | `a.b` → a1b, acb |
| `^` | 行开头 | `^192` 以 192 开头 |
| `$` | 行结尾 | `500$` 以 500 结尾 |
| `*` | 前一个字符 0 次或多次 | `ab*` → a, ab, abb |
| `[]` | 字符集，匹配其中一个 | `[0-9]` `[a-z]` |
| `[^]` | 取反字符集 | `[^0-9]` 非数字 |
| `\` | 转义符 | `\.` 匹配字面小数点 |

⚠️ **高频误区**：
- `*` 不是"任意字符"，是"前面字符 0 次或多次"。匹配任意字符串要用 `.*`。
- `.` 匹配任意字符，想匹配小数点必须写 `\.`。
- `^` 在 `[]` 外是行首，在 `[]` 内是取反。

---

### 扩展正则 ERE（推荐）

开启方式：`grep -E`、`sed -E`、awk 原生就是 ERE。

| 符号 | 含义 | 示例 |
|---|---|---|
| `+` | 前一个字符至少 1 次 | `[0-9]+` 一串数字 |
| `?` | 前一个字符 0 次或 1 次 | `colou?r` → color/colour |
| `\|` | 或 | `GET\|POST` |
| `()` | 分组捕获 | `(192\.168\.[0-9]+\.[0-9]+)` |
| `{n,m}` | 前一个字符出现 n~m 次 | `[0-9]{3}` 恰好3位数字 |

---

### 三个工具的正则差异（面试）

| 工具 | 正则类型 | 开启方式 |
|---|---|---|
| `grep` | BRE（默认）/ ERE | `grep -E` |
| `sed` | BRE（默认）/ ERE | `sed -E` |
| `awk` | **原生 ERE** | 不需要参数，直接写 |

```bash
# awk 直接写扩展正则，不用转义括号
awk '/(GET|POST)/{print $1}' access.log
```

---

### 实战正则

**IP 地址匹配**：
```
^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+
```

**5xx 状态码**：
```
5[0-9][0-9]$
```

**空行**：
```
^$
```

**开头有空白的行**：
```
^[ \t]+
```

---

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
