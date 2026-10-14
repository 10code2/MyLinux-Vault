# Linux 学习笔记库

> 个人 Linux / nginx 运维学习笔记 · Obsidian 知识库
> 本地路径：`D:\Obsidian\Linux` · 同步仓库：github.com/10code2/MyLinux-Vault

## 📁 目录结构

| 文件夹 | 内容主题 | 笔记 |
|---|---|---|
| `01-Linux基础` | 命令实战 / 错题 / 基础总结 | [[Linux命令系统实战笔记]] · [[Linux错题笔记-网络命令]] · [[Linux零基础到文本处理总结]] |
| `02-nginx` | nginx 学习笔记 | [[nginx静态网页学习笔记]] · [[nginx-负载均衡学习笔记]] · [[nginx-HTTPS学习笔记]] · [[nginx-502故障排查笔记]] |
| `03-排障案例` | 排障复盘与方法论 | [[nginx排障复盘]] · [[Linux常见故障排障方法]] |
| `04-系统巡检` | 巡检类工具笔记 | [[Linux系统资源全方位巡检]] |

## 🎯 学习路线（建议顺序）

1. **01-Linux基础**：零基础 → 命令实战 → 文本处理（grep/awk/sed）
2. **02-nginx**：静态托管 → 反向代理 → 负载均衡 → HTTPS → 安全防护
3. **03-排障案例**：用复盘沉淀"定位问题的思路"，比背命令更重要
4. **04-系统巡检**：日常巡检工具，作为运维日常工作手册

## 📝 维护约定

- 新笔记放入对应主题文件夹，命名格式：`主题-要点.md`（**避免** `#` 开头、全角标点、多余空格）
- 同一主题的更新版直接覆盖旧版，不要产生 `(1)` `(2)` 副本
- 定期 review：合并重复内容、标注过时知识点
- 重要结论写进笔记后，随手 `git add . && git commit -m "更新xxx" && git push`

## 🔄 GitHub 同步

```bash
# 首次
git init
git add .
git commit -m "init"
git remote add origin git@github.com:10code2/MyLinux-Vault.git
git push -u origin main

# 日常更新（一条链）
git add . && git commit -m "update" && git push
```
