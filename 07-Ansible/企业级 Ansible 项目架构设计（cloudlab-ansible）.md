---
tags: [ansible, 企业级, 架构设计, roles, 变量分层, vault, 自动化]
created: 2026-10-25
---
# 企业级 Ansible 项目架构设计（cloudlab-ansible）

> 相关笔记：[[Ansible 基础运维完整笔记]]｜[[Alpine Nginx 部署脚本完整解析]]｜[[Ansible Nginx 部署全链路踩坑运维笔记]]｜[[实操步骤]]
> 项目仓库：`D:\projects\cloudlab-ansible`（待推 GitHub）｜线上对照：`10code2/cloudlab`（手工 compose 版）

## 一、What：项目是什么

### 1.1 目标

把 CloudLab 自托管服务平台从「登录服务器手工敲 `docker-compose up`」升级为**声明式、可重复执行、跨环境批量部署**的 Ansible 项目：一条命令在任意主机上拉起整套系统，覆盖「系统基线 → 容器运行时 → 业务服务 → 监控体系」全链路。

### 1.2 覆盖范围

复刻现有 CloudLab 全套服务，共 10 个角色、3 套环境清单、4 个编排入口：

| 层次 | 角色 | 作用 |
|---|---|---|
| 系统基线 | `common` | 时区、基础工具、swap、内核参数、时间同步、可选运维账号 |
| 容器运行时 | `docker` | Docker 安装、镜像加速、外部共享网络、日志轮转 |
| 网关 | `npm` | Nginx Proxy Manager 统一入口、反代、HTTPS |
| 业务服务 | `vaultwarden` / `alist` / `halo` | 密码管理、云盘聚合、博客 |
| 监控 | `node-exporter` / `monitoring-prometheus` / `monitoring-grafana` / `uptime-kuma` | 采集 → 拉取 → 展示 → 可用性探活 |

### 1.3 与早期实验的关系

早期 `web-deploy` 是**单 playbook + 写死变量 + 一组节点**的练手版本；本项目是它的企业化重构：模块化复用、环境隔离、变量分层、敏感信息加密、预演校验、分批灰度。早期踩过的坑全部沉淀进设计（见第四节「历史踩坑如何反哺设计」）。

---

## 二、Why：为什么这样组织（企业级 vs 个人脚本）

> 核心问题：个人脚本能跑起来就行，企业级项目多出来的这些目录、角色、变量层，到底解决什么问题？

### 2.1 对比总览

| 维度 | 个人脚本（web-deploy 旧版） | 本项目（企业级做法） | 解决的真实问题 |
|---|---|---|---|
| 配置组织 | 单 playbook，任务平铺 | Role 模块化、职责单一、可跨环境复用 | 第二台机器、第二个项目想复用同一套基线时，不用复制粘贴 |
| 环境管理 | 只有一组节点 | dev / staging / prod 三套清单 | 防止在生产环境误跑实验配置；同一套代码适配不同环境 |
| 变量管理 | 硬编码、密码明文写在任务里 | 分层覆盖 + `ansible-vault` 加密 | 密码不进 Git；换环境只改变量不改代码 |
| 执行安全 | 直接落地执行 | `--check` 预演、`--diff` 看差异、`check.yml` 探活 | 上线前先看清会改什么；部署后验证服务真的可用 |
| 服务重启 | 每次无条件 restart | Handler：配置变更后才触发 reload | 避免无谓重启；保证「改了配置才重载」的因果闭环 |
| 任务粒度 | 整包一起跑 | `tags` 分批、`--limit` 单主机灰度 | 只改一个服务不用全量重启；排错时限定到单台主机 |
| 工程规范 | 无依赖管理 | `requirements.yml` 管 Collection、`ansible.cfg` 统一配置 | 换一台控制端能一键复现相同环境 |
| 幂等性 | 重复执行可能重复变更 | 模块状态参数 + `creates` 判断，重复执行不变更 | 配置可回归、可反复执行而不产生副作用 |

### 2.2 八个设计决策的原理剖析

**① 为什么拆成 Role，而不是继续写单 playbook？**
单 playbook 里任务、变量、模板混在一起，规模一大就出现复制粘贴和互相覆盖。Role 把「一个服务/一类基线」的 tasks、defaults、vars、templates、handlers 打包成独立单元：职责单一、可被多个 playbook 引用、可单独测试、便于团队分工。`common` 和 `docker` 是所有主机共享的基础角色，业务角色各自独立，互不耦合。

**② 为什么要三套 inventory？**
dev（WSL Docker 模拟节点，反复重建零成本）、staging（可选预发）、prod（真实云服务器）。环境隔离的本质是**把「主机差异」从代码里剥离**：同一套 playbook，dev 用模拟节点、prod 用真实 IP，差异只写在对应清单和变量里。这样既避免在 prod 上调试实验配置，也保证「在 dev 验证过的代码可以原样上 prod」。

**③ 为什么变量要分层，而不是全部写在 group_vars/all？**
分层遵循「单一事实来源」：公共默认进 role defaults（最低优先级，兜底）；跨环境公共项进 `group_vars/all`；某组服务特有进 `group_vars/<组>`；某台主机特有进 `host_vars/<主机>`；命令行 `-e` 临时覆盖最高。越具体的配置放越靠近主机的层，后续改一台机器不会污染全局。

**④ 为什么用 Handler，而不是任务末尾直接 restart？**
Handler 由 `notify` 触发，且**只有被 notify 的任务真的发生变更时才执行**。如果配置文件没变化，Handler 不跑，避免每次部署都重启服务。这同时满足幂等性：重复执行不产生无谓的重启动作。

**⑤ 为什么上线前要 `--check --diff`？**
`--check` 预演模式只计算会发生什么变更、不落地；`--diff` 同时打印配置文件前后差异。生产环境变更前先预演，可以在不影响业务的前提下确认「将要改的内容符合预期」，把风险挡在执行之前。

**⑥ 为什么敏感信息必须用 vault 加密？**
密码、管理员令牌、域名一旦明文提交 Git，即使后来删除也会残留在历史记录里，泄露面不可逆。`ansible-vault` 把 `group_vars/vault.yml` 整体加密，Git 里只存密文；执行时用 `--ask-vault-pass` 或密码文件解密，明文不落盘、不入库。

**⑦ 为什么需要 tags 和 limit？**
`tags` 把任务分组，`--tags halo` 只跑博客角色，不必全量部署；`--limit node1` 限定单台主机，用于灰度发布和单机排错。这是企业环境「分批上线、故障隔离」在 Ansible 里的对应手段。

**⑧ 为什么强调幂等？**
幂等 = 同一个 playbook 执行一次和执行十次，最终系统状态一致、第二次执行 `changed` 数量为 0。实现方式：优先用自带状态判断的模块（`state: present/absent`），文件类任务用 `creates/removes` 判断，避免裸 `shell` 无条件执行命令。幂等让配置可以安全回归、定时重复执行，也是自动化可信的前提。

---

## 三、Where：目录结构与变量落点

### 3.1 目录树

```
cloudlab-ansible/
├── ansible.cfg                  # 全局配置：清单路径、SSH 复用、日志、角色路径
├── requirements.yml             # Collection 依赖（community.docker / community.general / ansible.posix）
├── inventory/
│   ├── dev.ini                  # 开发环境：WSL Docker 节点（2201/2202/2203）
│   ├── staging.ini              # 预发环境（可选）
│   └── prod.ini                 # 生产环境：真实云服务器
├── group_vars/
│   ├── all.yml                  # 公共变量：时区、Docker 加速、目录、资源限制
│   ├── webservers.yml           # Web 服务分组：NPM / Vaultwarden / Alist / Halo
│   ├── monitoring.yml           # 监控分组：Prometheus / Grafana / Exporter / Kuma
│   └── vault.yml                # 敏感变量（vault 加密）
├── host_vars/prod-01.yml        # 单机覆盖（更严格的内存上限等）
├── playbooks/
│   ├── site.yml                 # 总入口：基线 + Docker + 业务 + 监控
│   ├── init.yml                 # 仅系统基线 + Docker
│   ├── deploy-services.yml     # 仅业务服务
│   └── check.yml                # 部署后健康检查
├── roles/                       # 10 个角色，每个含 tasks / defaults / handlers / templates
├── files/                       # 需复制到被控端的静态文件
└── docs/cloud-service-selection.md   # 多环境云服务器选型调研
```

### 3.2 变量优先级链（由低到高，后者覆盖前者）

```
role defaults  <  group_vars/all  <  group_vars/<组>  <  host_vars/<主机>
            <  playbook vars  <  命令行 -e 覆盖
```

- 兜底默认 → 跨环境公共 → 分组特有 → 单机特有 → playbook 内联 → 命令行临时覆盖。
- 排障时若变量「不生效」，先按这条链判断是不是被更上层覆盖了。

### 3.3 环境分组与服务端口对照

| 分组 | 服务 | 角色 | 主机端口 |
|---|---|---|---|
| webservers | NPM 网关 | `npm` | 80 / 443 / 81 |
| webservers | Vaultwarden | `vaultwarden` | 8080 |
| webservers | Alist | `alist` | 5244 |
| webservers | Halo 博客 | `halo` | 8090 |
| monitoring | Node Exporter | `node-exporter` | 9100 |
| monitoring | Prometheus | `monitoring-prometheus` | 9090 |
| monitoring | Grafana | `monitoring-grafana` | 3000 |
| monitoring | Uptime Kuma | `uptime-kuma` | 3001 |

---

## 四、How：落地步骤与关键原理

### 4.1 分阶段任务清单（自己动手实现）

- [ ] **阶段 0**：`git init`，`ansible-galaxy collection install -r requirements.yml`，确认 `ansible.cfg` 生效
- [ ] **阶段 1**：补齐 `inventory/dev.ini` 节点信息，完善 `group_vars` 变量
- [ ] **阶段 2**：跑通 `common` 角色，在 dev 三节点验证时区/工具/swap/内核参数
- [ ] **阶段 3**：跑通 `docker` 角色，确认外部网络 `cloudlab-net` 创建成功
- [ ] **阶段 4**：逐角色实现并逐个验证：`npm` → `vaultwarden` → `alist` → `halo` → `node-exporter` → `monitoring-prometheus` → `monitoring-grafana` → `uptime-kuma`
- [ ] **阶段 5**：`ansible-vault encrypt group_vars/vault.yml`，敏感信息不进 Git
- [ ] **阶段 6**：`--check --diff` 全量预演 → 正式执行 → `check.yml` 探活
- [ ] **阶段 7**：补全 README，推送 GitHub，沉淀笔记

> `common`、`docker`、`npm` 三个角色已给出完整参考实现；其余角色 `tasks/main.yml` 内已用 TODO 注释列出步骤、模块名与注意事项，参照 `npm` 角色的「建目录 → 渲染模板 → compose 启动 → 探活 → handler」四段式补齐即可。

### 4.2 命令速查

```bash
# 安装依赖
ansible-galaxy collection install -r requirements.yml

# 连通性检查（不变更）
ansible all -i inventory/dev.ini -m ping

# 预演 + 看差异（不落地）
ansible-playbook playbooks/site.yml -i inventory/dev.ini --check --diff

# 全量执行 dev（交互输入 vault 密码）
ansible-playbook playbooks/site.yml -i inventory/dev.ini --ask-vault-pass

# 只跑系统基线
ansible-playbook playbooks/init.yml -i inventory/dev.ini --tags baseline

# 只部署某个服务
ansible-playbook playbooks/deploy-services.yml -i inventory/dev.ini --tags halo

# 限定单台主机灰度/排错
ansible-playbook playbooks/site.yml -i inventory/dev.ini --limit node1

# 部署后健康检查
ansible-playbook playbooks/check.yml -i inventory/dev.ini
```

### 4.3 原理深挖一：Handler 的执行时机（历史深坑）

早期实验踩过：任务显示 `changed`，但服务没重载，验证任务拿到旧配置直接 404。根因与对策：

1. Handler 在**整个 play 末尾**才统一执行，不是 notify 后立即跑；
2. 失败主机上已 notify 的 Handler **不会执行**；
3. 验证类任务若必须在重载后跑，前面要加 `meta: flush_handlers` 提前触发已注册的 Handler，形成「变更 → 重载 → 验证」闭环；
4. 本项目每个服务角色都配了独立 Handler（`restart <service>`），配置模板变更时才 `notify`，避免无条件重启。

### 4.4 原理深挖二：幂等的实现边界

- 优先用状态型模块：`package: state=present`、`file: state=directory`、`docker_compose: state=present`，模块自带是否已满足的判断；
- 一次性命令类任务用 `creates`（已存在则跳过）或 `removes`（不存在则跳过）包裹；
- 避免裸 `shell: docker-compose up -d` 这类每次都执行、无法判断状态的写法；
- 每次部署后看 `PLAY RECAP`：`changed` 应只集中在真的需要变更的任务上；二次执行 `changed=0` 才说明幂等达标。

### 4.5 原理深挖三：跨发行版兼容

`common` 角色用 `ansible_os_family` 区分 Debian 系（`apt`、`chrony`）与 RedHat 系（`yum/dnf`、`chronyd`），同一套任务在两类系统上选择对应包名与服务名。这也是企业环境「一套代码纳管多发行版主机」的基本要求。

### 4.6 历史踩坑如何反哺设计

| 早期踩坑 | 本项目对应的设计动作 |
|---|---|
| Handler 不及时触发导致验证失败 | 每个服务配独立 Handler；关键链路 `flush_handlers` |
| 全新 Halo 根路径 404，误判服务挂了 | `check.yml` 探活打 `/console`（302 登录跳转）而非 `/` |
| Kuma 用公网 IP 监控自身出现 NAT hairpin 误报 | 文档约定监控目标优先用容器名/内网地址 |
| 2G 内存 Java 服务 OOM | `halo` 角色强制 `Xms/Xmx` 堆限制；`common` 提供 swap |
| 镜像拉取超时 | `docker` 角色统一配置镜像加速源与日志轮转 |
| 密码/令牌散落在各 compose 文件 | 全部收敛进 `vault.yml` 加密管理 |

### 4.7 已知约束（落地前必须知道）

1. **CentOS 7 纳管受限**：ansible-core 2.19 已移除 `yum` 模块，且被控端 Python 3.6 不满足要求；纳管前需把控制端降到 2.12，或在被控端装 Python 3.8+ 并指定 `ansible_python_interpreter`。
2. **2 核 2G 内存紧张**：JVM 堆、Prometheus/Grafana 都要设内存上限，swap 兜底。
3. **未备案域名 80/443 被运营商阻断**：国内节点证书走 NPM + DNS-Challenge，公网访问受备案限制。
4. 云服务器选型对比见 `docs/cloud-service-selection.md`：多节点联调用 WSL Docker 零成本，真实节点选 2 核 2G 轻量，1 核 1G 免费层跑不动整套服务。

---

## 五、核心结论速查

- 企业级自动化的本质：**代码即配置、环境即变量、变更可预演、结果可回归**。
- Role 解决复用与职责，inventory 解决环境隔离，变量分层解决差异外化，vault 解决安全，Handler 解决变更闭环，check mode + 探活解决可信度。
- 判断 playbook 是否合格：二次执行 `changed=0`、`check.yml` 全部探活通过、敏感变量不出现在 Git 明文里。
- 早期个人实验解决「能跑起来」，本项目解决「别人接手、换台机器、换个环境照样跑起来且不闯祸」。

## 下一步计划

- [ ] 按 4.1 阶段清单逐角色实现，每完成一个角色在 dev 验证
- [ ] 全量 `--check` 预演通过后执行部署，`check.yml` 探活全绿
- [ ] 加密 vault 变量并推送 GitHub，补全 README 架构图与徽章
- [ ] 把 `web-deploy` 旧实验归档，README 中指向本项目
