---
tags: [ansible, 排障, ssh, docker, 踩坑]
created: 2026-10-24
---
## 一、故障总览

> 相关笔记：[[Ansible Nginx 部署全链路踩坑运维笔记]]（坑点 1 为本篇摘要）｜[[Ansible 基础运维完整笔记]]

### 1.1 典型报错现象

Ansible 执行命令后返回 `UNREACHABLE`，核心错误信息：

```
Failed to connect to the host via ssh: ssh: connect to host localhost port 2201: Connection refused
```

- `UNREACHABLE`：Ansible 无法触达目标节点，任务无法下发
- `Connection refused`：请求到达了目标地址和端口，但端口上没有程序监听，被系统直接拒绝

### 1.2 核心本质

**这不是 Ansible 配置问题，而是底层 SSH 连接本身不通。** Ansible 基于 SSH 协议工作，底层 SSH 连不通，Ansible 必然无法执行任何操作。排障必须从基础设施层开始，而非上来就修改 Ansible 配置。

### 1.3 适用环境

本地 Docker 容器模拟 Ansible 被管节点，通过端口映射（2201/2202/2203）暴露 SSH 服务，控制端为本地 WSL / Linux 主机。

---

## 二、排障核心原则

**自底向上，逐层验证**：先确认底层资源正常，再检查网络端口，再验证协议服务，最后排查上层工具配置。

> 和系统性能排查思路一致：先定性后深挖，先底层后上层，避免把简单问题复杂化。

---

## 三、标准四步排查流程

### 第 1 步：确认容器运行状态（资源层）

**最常见的根因：容器没启动**

```
# 查看所有容器状态，含已停止的
docker ps -a
```

**判断标准**：

- `STATUS` 为 `Up xxx` → 容器运行中
- `STATUS` 为 `Exited (xxx)` → 容器已停止，所有端口映射失效

**常见停止原因**：

- 忘记执行 `docker start`
- Docker 服务重启后容器未自动拉起
- 容器内部进程异常退出（如 sshd 启动失败）

### 第 2 步：验证本地端口监听（网络层）

容器运行也可能端口没起来，直接从系统层面确认：

```
# 查看所有TCP监听端口，过滤目标端口
ss -tlnp | grep -E '2201|2202|2203'
```

- 有输出：端口已正常监听
- 无输出：容器虽在运行，但内部 SSH 服务没启动，或端口映射配置错误

### 第 3 步：纯 SSH 直连测试（协议层）

绕开 Ansible，直接用 SSH 命令测试，排除工具干扰：

```
ssh -p 2201 -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -i /root/.ssh/id_ed25519 root@localhost
```

- 能正常登录 → 底层 SSH 完全正常，问题在 Ansible 配置层
- 同样报错 → 问题在底层，和 Ansible 无关，继续向内排查

### 第 4 步：排查 Ansible 配置（工具层）

底层全部正常仍失败，再检查配置：

1. **inventory 路径是否正确**：避免相对路径、目录误当文件
2. **主机参数是否正确**：`ansible_host`、`ansible_port`、密钥路径是否匹配
3. **Python 解释器**：目标节点 Python 路径是否正确
4. **清单合法性校验**：
    
    ```
    ansible-inventory -i inventory --list
    ```
    

---

## 四、四大高频故障场景与修复

### 场景 1：容器未启动（最高频）

- **现象**：`docker ps -a` 显示 `Exited` 状态，端口无监听
- **原因**：忘记启动容器、Docker 重启后容器未自动拉起
- **修复**：
    
    ```
    # 启动所有节点容器
    docker start node1 node2 node3
    ```
    

### 场景 2：容器运行，但 sshd 进程退出

- **现象**：容器状态 `Up`，但端口无监听，`docker logs` 显示 sshd 报错退出，退出码 255
- **常见原因**：
    1. SSH 主机密钥缺失或损坏
    2. 非正常关机导致 `sshd.pid` 残留，sshd 误以为已有进程在运行
    3. 密钥文件权限不符合 sshd 安全要求
- **修复（一次性重建，内置容错）**：
    
    ```
    # 清理旧容器
    docker rm -f node1 node2 node3
    
    # 重建 node1（另外两台同理替换端口和名称）
    docker run -d --name node1 --restart=always -p 2201:22 ansible-node bash -c "
    mkdir -p /var/run/sshd
    rm -f /var/run/sshd.pid
    ssh-keygen -A
    chmod 600 /etc/ssh/ssh_host_* 2>/dev/null
    /usr/sbin/sshd -D
    "
    ```
    
    > 启动前置逻辑：清 pid → 生成主机密钥 → 修复权限 → 启动 sshd，覆盖 90% 的启动失败原因
    

### 场景 3：端口映射错误

- **现象**：容器正常运行，内部 sshd 也正常，但本地端口连不上
- **原因**：重建容器时忘记指定 `-p` 参数，端口未映射到宿主机
- **修复**：删除容器重新创建，明确指定端口映射 `-p 宿主机端口:22`

### 场景 4：Ansible inventory 配置错误

- **现象**：SSH 直连正常，Ansible 连不上
- **常见错误**：
    - 路径错误：inventory 是目录不是文件，或相对路径不匹配
    - 端口/地址写错：`ansible_port` 与实际映射端口不一致
    - 密钥路径错误：私钥文件不存在或权限不对
- **修复**：核对 inventory 参数，使用绝对路径指定清单文件

---

## 五、长效优化方案

### 5.1 容器自动重启

给所有节点配置重启策略，Docker 服务恢复后容器自动拉起：

```
docker update --restart=always node1 node2 node3
```

- `always`：Docker 启动时自动启动容器，容器意外退出也自动重启
- 彻底解决「忘记启动容器」的人为失误

### 5.2 启动命令内置容错

镜像或启动命令中加入前置清理逻辑，确保异常关机后仍能正常启动：

```
rm -f /var/run/sshd.pid
mkdir -p /var/run/sshd
ssh-keygen -A
```

从源头解决 pid 残留、密钥缺失导致的 sshd 启动失败。

### 5.3 生产级 Ansible 配置规范

1. inventory 文件一律使用**绝对路径**，避免工作目录变化导致失效
2. 连接参数集中放在组变量，不重复书写
3. 实验环境可临时关闭主机密钥校验，生产环境务必开启
4. 明确指定 Python 解释器路径，避免多版本环境报错

---

## 六、易错点与避坑指南

1. **停止的容器也有端口映射条目**`docker ps -a` 能看到端口映射，不代表端口正在监听；只有容器运行中，映射才真正生效。
2. **Connection refused ≠ 网络不通** 网络不通会报超时（timeout）；refused 是端口上没有服务监听，90% 是目标进程未启动。
3. **不要上来就改 Ansible 配置** 90% 的连接失败问题都出在底层，先查容器、端口、SSH，最后再查 Ansible 配置，避免浪费时间。
4. **退出码 255 是 sshd 专属信号** 容器以 255 退出，基本可以直接定位是 SSH 服务启动失败，优先查密钥、pid 文件、权限。
5. **容器非正常关机必留坑** Docker 强制重启、WSL 断电、宿主机死机，都可能导致 sshd.pid 残留，下次启动直接失败，启动命令加清理逻辑最稳妥。

---

## 七、排障速记口诀

容器先看运行态，再查端口在不在， SSH 直连先验证，最后再把配置改。