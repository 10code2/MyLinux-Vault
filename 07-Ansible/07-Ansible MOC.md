---
tags: [ansible, MOC, 索引]
created: 2026-10-24
---
# 07-Ansible 知识地图（MOC）

> 本目录围绕「Docker Alpine 容器作为被管节点，Ansible 批量部署 Nginx」这一主线实验展开，按「理论 → 解析 → 实操 → 踩坑复盘」组织。

## 学习路径

1. [[Ansible 基础运维完整笔记]] — 理论底座：架构、inventory、幂等性、Ad-Hoc、Playbook 入门
2. [[Alpine Nginx 部署脚本完整解析]] — 逐任务拆解 deploy.yml，含伪幂等分析与优化版
3. [[实操步骤]] — 动手枢纽：inventory + 最终版 deploy.yml（已含 `meta: flush_handlers`）
4. 踩坑复盘三篇（按问题层次）：
   - [[Ansible 节点连接失败排障完整运维笔记]] — 连接层：容器/sshd/端口/配置四步排障
   - [[Ansible Nginx 部署全链路踩坑运维笔记]] — 部署层：5 个坑 + Handler 执行时机深坑
   - [[Ansible Nginx 部署：IPv6 解析与精简镜像工具缺失踩坑笔记]] — 验证层：工具缺失 + IPv6 解析造成的双重假象
5. 工程化进阶：[[企业级 Ansible 项目架构设计（cloudlab-ansible）]] — 从单 playbook 重构为多环境、多角色、变量分层、vault 加密的企业级项目

## 核心结论速查

- 任务执行成功 ≠ 服务真实可用；文件幂等 ≠ 配置已生效
- 排障自底向上：容器状态 → 端口监听 → 进程 → 配置 → Ansible
- 验证类任务前必须 `meta: flush_handlers`，形成「变更 → 重载 → 验证」闭环
- 精简发行版不预装运维工具、不创建默认目录，不要套用全量系统的行为预期
- Alpine apk 的 Nginx 默认根目录为 `/var/lib/nginx/html`（旧版本/第三方镜像可能为 `/usr/share/nginx/html`）
- `localhost ≠ 127.0.0.1`：解析优先走 IPv6，服务建议双栈监听
- 企业级自动化：Role 复用、环境隔离、变量分层、vault 加密、check 预演、Handler 变更闭环、幂等回归（详见 [[企业级 Ansible 项目架构设计（cloudlab-ansible）]]）

## 下一步计划

- [x] 将 deploy.yml 重构为 Role（已落地为 cloudlab-ansible 项目骨架，待逐角色填实现）
- [ ] 逐角色实现 cloudlab-ansible，dev 环境全量预演 + 探活通过
- [ ] 上传 GitHub 沉淀
