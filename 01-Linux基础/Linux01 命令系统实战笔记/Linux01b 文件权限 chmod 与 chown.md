---
title: Linux01b 文件权限 chmod 与 chown
created: 2026-10-16
updated: 2026-10-24
status: 待复习
tags: [linux, 基础]
---

> 权限模型 + chmod/chown 实战 · 拆分自 [[Linux01 命令系统实战笔记]]

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


## 🔗 相关笔记

- [[Linux01a 文件查看与查找命令]]
- [[Linux运维面试题整理]]
