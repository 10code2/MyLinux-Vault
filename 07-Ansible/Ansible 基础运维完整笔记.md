---
tags: [ansible, 基础, inventory, 幂等性, ad-hoc]
created: 2026-10-24
---
## 一、Ansible 基础认知

> 相关笔记：[[实操步骤]]｜[[Alpine Nginx 部署脚本完整解析]]｜[[Ansible 节点连接失败排障完整运维笔记]]

### 1.1 核心架构

Ansible 是**无客户端**的批量运维工具，分为两类角色：

|角色|说明|安装要求|
|---|---|---|
|**控制节点**|执行 `ansible` 命令的机器，下发批量任务|必须安装 Ansible 环境|
|**被管节点（Node）**|被批量操作的目标机器，即 inventory 中定义的主机|无需安装 Ansible，仅需满足基础连接条件|

### 1.2 被管节点（Node）的类型与准入条件

#### 支持的节点类型

- Linux 物理机 / 虚拟机 / 云服务器（最主流，默认 SSH 协议）
- Docker / LXC 容器（开启 SSH 服务即可，常用于实验环境）
- 网络设备（交换机、路由器、防火墙，支持 SSH）
- Windows 服务器（通过 WinRM 协议连接）
- 云平台资源（ECS、负载均衡、存储等，调用云厂商 API 管理）

#### 成为被管节点的两个核心条件

1. **网络连通**：控制节点可通过对应协议访问目标节点（Linux 默认 22 端口 SSH）
2. **Python 环境**：节点需有 Python 解释器（2.7 / 3.5+），Ansible 模块依赖其执行

> `ansible ping` 模块返回 `pong`，等价于同时验证「SSH 连通 + Python 环境正常」两个前提。

### 1.3 无客户端架构优势

- 被管节点无需部署 Agent，减少维护成本
- 基于 SSH 协议，通用性强，多数系统原生支持
- 控制节点下发任务时才推送模块，执行完成后清理，无常驻进程

---

## 二、Inventory 主机清单详解

### 2.1 核心作用

Inventory 是 Ansible 的「主机通讯录」，定义了所有被管节点的地址、连接方式、分组关系，是所有批量操作的基础。

### 2.2 两种主流格式

#### 1）INI 格式（入门首选，直观简洁）

适合中小规模环境，语法简单，可读性强。

**最简示例**

```
node1 ansible_host=127.0.0.1 ansible_port=2201
node2 ansible_host=127.0.0.1 ansible_port=2202
node3 ansible_host=127.0.0.1 ansible_port=2203
```

**分组写法（生产环境必用）** 按业务角色分组，实现精准批量操作：

```
[web]
node1 ansible_host=127.0.0.1 ansible_port=2201
node2 ansible_host=127.0.0.1 ansible_port=2202

[db]
node3 ansible_host=127.0.0.1 ansible_port=2203
```

#### 2）YAML 格式（进阶，结构清晰）

适合复杂环境、多层分组嵌套，与 Playbook 格式统一：

```
all:
  children:
    web:
      hosts:
        node1:
          ansible_host: 127.0.0.1
          ansible_port: 2201
        node2:
          ansible_host: 127.0.0.1
          ansible_port: 2202
    db:
      hosts:
        node3:
          ansible_host: 127.0.0.1
          ansible_port: 2203
```

### 2.3 分组与变量

#### 主机分组

- `all`：内置分组，包含清单中所有主机
- 自定义分组：如 `[web]`、`[db]`，可按业务、环境、地域划分
- 分组嵌套：通过 `:children` 实现组中组，适合多层级架构
    
    ```
    [all_server:children]
    web
    db
    ```
    

#### 变量层级

1. **主机变量**：写在单台主机后，仅对该主机生效
2. **组变量**：`[组名:vars]`，组内所有主机共用
3. **全局变量**：`[all:vars]`，所有主机共用

**完整带变量示例（适配本地容器实验环境）**

```
[web]
node1 ansible_host=localhost ansible_port=2201
node2 ansible_host=localhost ansible_port=2202

[db]
node3 ansible_host=localhost ansible_port=2203

[all:vars]
ansible_user=root
ansible_ssh_private_key_file=/root/.ssh/id_ed25519
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_common_args='-o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no'
```

### 2.4 常用内置连接变量

|变量名|作用|
|---|---|
|`ansible_host`|目标主机真实 IP / 域名|
|`ansible_port`|SSH 连接端口，默认 22|
|`ansible_user`|SSH 登录用户名|
|`ansible_ssh_private_key_file`|SSH 私钥文件路径|
|`ansible_python_interpreter`|指定目标机 Python 解释器路径|
|`ansible_ssh_common_args`|追加 SSH 命令额外参数|

> 实验环境常用的两个 SSH 参数说明：
> 
> - `StrictHostKeyChecking=no`：关闭主机指纹校验，首次连接无交互提示
> - `UserKnownHostsFile=/dev/null`：不保存主机指纹，避免容器重建后指纹冲突 ⚠️ 仅适用于本地实验，生产环境严禁开启，存在中间人攻击风险。

### 2.5 快速生成：Here Document 写法

通过 Shell 内嵌文档一键生成 inventory 文件，适合脚本化快速搭建环境：

```
cat > inventory <<'EOF'
[web]
node1 ansible_host=localhost ansible_port=2201
node2 ansible_host=localhost ansible_port=2202

[db]
node3 ansible_host=localhost ansible_port=2203

[all:vars]
ansible_user=root
ansible_ssh_private_key_file=/root/.ssh/id_ed25519
ansible_ssh_common_args='-o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no'
EOF
```

**关键细节**：`<<'EOF'` 加单引号，禁止 Shell 变量展开，保证内容原样写入；结尾 `EOF` 必须顶格，不能有缩进。

### 2.6 清单合法性校验

```
# 解析并输出完整清单结构，验证格式是否合法
ansible-inventory -i inventory --list
```

---

## 三、核心特性：幂等性

### 3.1 什么是幂等性

**同一个操作，执行 1 次和执行 N 次，最终系统状态完全一致，不会产生重复副作用。** 简单说：不会因为多执行一遍，就把系统搞坏、重复安装、重复追加内容。

### 3.2 幂等 vs 非幂等 对比

|类型|示例|执行效果|
|---|---|---|
|非幂等|`echo "123" >> test.txt`|执行 1 次追加 1 行，执行 10 次追加 10 行|
|幂等|Ansible `copy` 模块写入文件|内容一致则不做任何修改，内容不一致才更新|

### 3.3 `changed` 字段的含义

Ansible 执行结果中的 `changed` 是幂等性的直接体现：

- `changed: true`：本次执行**修改了**目标主机的状态
- `changed: false`：本次执行**未修改**目标主机，状态与预期一致

示例：

- `ping` 模块仅测试连通，不修改系统 → 永远 `changed: false`
- `yum` 模块安装软件，第一次安装返回 `true`，已安装后再执行返回 `false`

### 3.4 常见模块幂等性分类

|模块类型|代表模块|幂等性|
|---|---|---|
|专用业务模块|ping、yum、service、copy、file、firewalld|✅ 天然幂等，自动判断是否需要执行|
|通用命令模块|command、shell、raw|❌ 不保证幂等，效果取决于命令本身，默认执行成功即返回 `CHANGED`|

> 运维规范：能使用专用模块就不用 shell/command，更稳定、更符合幂等原则。

### 3.5 幂等性的运维价值

1. **安全可靠**：脚本重复执行、任务重试，不会破坏系统状态
2. **便于审计**：`changed=true` 代表真实发生了变更，方便追溯
3. **批量稳定**：多节点分批执行、中途失败重试，不会产生不一致
4. **配置一致**：多次执行后，所有节点状态始终保持统一

---

## 四、Ad-Hoc 临时命令常用操作

Ad-Hoc 是 Ansible 的单次批量命令模式，适合临时操作、快速验证。

### 4.1 基础语法

```
ansible <目标主机/组> -i <清单路径> -m <模块名> -a <模块参数>
```

### 4.2 高频模块与示例

#### 1）ping：连通性测试

```
ansible all -i inventory -m ping
```

- 作用：验证 Ansible 与目标节点的 SSH 连通性 + Python 环境
- 返回 `pong` 代表正常，比普通 ICMP ping 更有说服力

#### 2）command / shell：执行命令

```
# 执行简单命令（不支持管道、重定向）
ansible all -i inventory -m command -a "uptime"

# 支持管道、变量等复杂 Shell 语法
ansible all -i inventory -m shell -a "free -h | grep Mem"
```

> 两者均为非幂等模块，默认返回 `CHANGED`。

#### 3）yum：软件包管理

```
# 安装软件
ansible web -i inventory -m yum -a "name=nginx state=present"

# 卸载软件
ansible web -i inventory -m yum -a "name=nginx state=absent"
```

- 幂等模块，自动判断软件是否已安装

#### 4）service：服务管理

```
# 启动服务并设置开机自启
ansible web -i inventory -m service -a "name=nginx state=started enabled=yes"

# 重启服务
ansible web -i inventory -m service -a "name=nginx state=restarted"
```

#### 5）copy：文件批量分发

```
# 写入内容到目标文件
ansible web -i inventory -m copy -a 'content="<h1>Web Node</h1>" dest=/usr/share/nginx/html/index.html'
```

- 幂等模块，内容一致则不修改

#### 6）firewalld：防火墙管理

```
# 放行 80 端口，永久生效并立即生效
ansible web -i inventory -m firewalld -a "port=80/tcp permanent=yes state=enabled immediate=yes"
```

---

## 五、常见报错与排查

### 5.1 inventory 解析失败

**报错信息**

```
[WARNING]: Unable to parse xxx as an inventory source
[WARNING]: No inventory was parsed, only implicit localhost is available
```

**常见原因与解决**

1. **路径错误 / 文件不存在**（最高频）
    - 现象：相对路径执行报错，绝对路径正常
    - 解决：使用绝对路径指定 inventory，或确认文件在当前工作目录
2. **路径指向了目录而非文件**
    - 现象：`ls` 显示 `total 0`，inventory 是个空目录
    - 解决：指定到具体的清单文件，而非目录
3. **格式错误**
    - 常见：全角符号、变量名拼写错误、分组语法错误
    - 排查：用 `ansible-inventory -i inventory --list` 精准定位错误行

### 5.2 主机不可达（UNREACHABLE）

- 检查目标 IP、端口是否正确
- 验证 SSH 免密登录是否正常
- 确认目标节点防火墙、安全组放行 SSH 端口
- 检查目标节点 SSH 服务是否正常运行

### 5.3 命令执行失败（FAILED）

- 检查命令语法是否正确
- 确认目标节点是否安装了对应命令
- 权限不足：添加 `become: yes` 提权
- Python 环境异常：通过 `ansible_python_interpreter` 指定正确路径

---

## 六、实战：批量部署 Nginx Web 服务

### 6.1 环境说明

- 控制节点：本地 Ansible 环境
- 被管节点：`web` 组 2 台节点（node1、node2），本地容器模拟
- 目标：批量安装 Nginx、启动服务、部署自定义首页、放行防火墙

### 6.2 方式一：Ad-Hoc 分步部署

适合快速测试、临时操作。

```
# 1. 安装 Nginx
ansible web -i /root/ansible-lab/inventory -m yum -a "name=nginx state=present"

# 2. 启动服务并设置开机自启
ansible web -i /root/ansible-lab/inventory -m service -a "name=nginx state=started enabled=yes"

# 3. 部署自定义首页（显示节点名）
ansible web -i /root/ansible-lab/inventory -m copy -a 'content="<h1>Web Server: {{ inventory_hostname }}</h1>" dest=/usr/share/nginx/html/index.html'

# 4. 防火墙放行 80 端口（容器环境可跳过）
ansible web -i /root/ansible-lab/inventory -m firewalld -a "port=80/tcp permanent=yes state=enabled immediate=yes"
```

### 6.3 方式二：Playbook 剧本部署（生产推荐）

将所有步骤整合为 YAML 剧本，可复用、可追溯、全程幂等。

新建 `deploy_nginx.yml`：

```
---
- name: 批量部署 Nginx Web 服务
  hosts: web
  become: yes
  tasks:

    - name: 1. 安装 Nginx 软件包
      yum:
        name: nginx
        state: present

    - name: 2. 启动 Nginx 并设置开机自启
      service:
        name: nginx
        state: started
        enabled: yes

    - name: 3. 部署自定义首页
      copy:
        content: "<h1>Web Server: {{ inventory_hostname }}</h1>"
        dest: /usr/share/nginx/html/index.html

    - name: 4. 防火墙放行 80 端口
      firewalld:
        port: 80/tcp
        permanent: yes
        state: enabled
        immediate: yes
      ignore_errors: yes   # 容器无防火墙时忽略报错
```

执行剧本：

```
ansible-playbook -i /root/ansible-lab/inventory deploy_nginx.yml
```

### 6.4 结果验证

```
# 批量查看服务状态
ansible web -i inventory -m command -a "systemctl status nginx"

# 批量本地 curl 验证页面
ansible web -i inventory -m command -a "curl -s http://127.0.0.1"
```

---

## 七、最佳实践与避坑指南

1. **路径规范**：生产环境一律使用绝对路径指定 inventory，避免相对路径依赖工作目录
2. **模块选择**：优先使用官方专用模块，减少 shell/command 的使用
3. **分组管理**：按业务、环境、角色合理分组，避免全部主机混在一起
4. **变量集中**：连接参数、公共配置统一放在组变量 / 全局变量，减少重复配置
5. **幂等设计**：编写任务时始终考虑幂等性，避免重复执行产生副作用
6. **安全规范**：
    - 生产环境禁止关闭主机密钥校验
    - 使用密钥登录，避免明文密码
    - 遵循最小权限原则，合理配置 sudo 提权
7. **排障顺序**：先检查 inventory 配置 → 再验证 SSH 连通 → 最后排查模块与命令问题