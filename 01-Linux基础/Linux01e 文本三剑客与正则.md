---
title: Linux01e 文本三剑客与正则
created: 2026-10-16
updated: 2026-10-24
status: 待复习
tags: [linux, 基础]
---

> grep/cut/sort/uniq/tr + awk 详解 + sed 详解 + 正则表达式 · 拆分自 [[Linux01 命令系统实战笔记]]

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


## 🔗 相关笔记

- [[Linux02 零基础到文本处理总结]]
- [[nginx-502故障排查笔记]]
