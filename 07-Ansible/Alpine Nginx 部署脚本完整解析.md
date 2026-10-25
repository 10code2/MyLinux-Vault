---
tags: [ansible, playbook, nginx, alpine]
created: 2026-10-24
---
# Ansible Playbook：Alpine Nginx 部署脚本完整解析

> 相关笔记：[[实操步骤]]｜[[Ansible Nginx 部署全链路踩坑运维笔记]]｜[[Ansible Nginx 部署：IPv6 解析与精简镜像工具缺失踩坑笔记]]

## 一、文件生成方式与整体定位

### 1.1 生成语法说明

你使用 Shell Here Document 一键生成 Playbook 文件：

```
cat > deploy.yml <<'EOF'
...
EOF
```

- 作用：将中间的 YAML 内容一次性写入当前目录的 `deploy.yml` 文件
- 关键细节：`<<'EOF'` 加单引号，**禁止 Shell 变量展开**，保证 Ansible 的 `{{ inventory_hostname }}` 变量不会被本地 Shell 解析为空，原样写入配置。和之前生成 inventory 的写法逻辑完全一致。

### 1.2 整体功能

这是一个**针对 Alpine Linux 系统（apk 包管理器）**的 Nginx Web 服务批量部署 Playbook，目标是在 `web` 分组的所有节点上，自动化完成「安装软件 → 创建目录 → 部署首页 → 启动服务」全流程。 适配场景：轻量 Docker 容器实验环境（Alpine 镜像体积小，常用来做 Ansible 被管节点）。

---

## 二、Play 头部配置拆解

```
---
- name: 部署 Nginx Web 服务
  hosts: web
  become: true
  tasks:
```

|字段|作用|
|---|---|
|`---`|YAML 文件标准开头标记|
|`name`|Play 可读名称，执行时用于日志展示与区分|
|`hosts: web`|目标主机范围，仅操作 inventory 中 `web` 分组的节点|
|`become: true`|执行任务时自动提权（默认切换到 root），安装软件、修改系统目录必须开启|
|`tasks`|任务列表，按从上到下顺序串行执行，失败自动中止|

---

## 三、逐任务深度解析

### 任务 1：安装 Nginx 软件包

```
- name: 安装 nginx
  apk:
    name: nginx
    state: present
    update_cache: yes
```

- **模块**：`apk`，Alpine Linux 专属包管理模块，等价于 CentOS 环境的 `yum` 模块
- **参数拆解**：
    - `name: nginx`：指定安装的软件包名
    - `state: present`：确保软件包存在，不存在则安装
    - `update_cache: yes`：安装前先更新本地软件源缓存（类似 `apk update`），避免源过期导致安装失败
- **幂等性**：✅ 天然幂等。已安装的软件不会重复安装，重复执行返回 `changed: false`。

### 任务 2：创建网站根目录

```
- name: 创建网站根目录
  file:
    path: /var/www/html
    state: directory
    owner: nginx
    group: nginx
    mode: '0755'
```

- **模块**：`file`，Ansible 通用文件/目录管理模块
- **参数拆解**：
    - `path: /var/www/html`：目标目录路径
    - `state: directory`：确保目标是目录，不存在则创建，存在则校验属性
    - `owner/group`：设置目录的属主、属组为 nginx 用户，保证 Nginx 进程有读取权限
    - `mode: '0755'`：目录权限，所有者读写执行，其他用户读执行
- **幂等性**：✅ 天然幂等。目录存在且属性一致则不做任何修改。

### 任务 3：部署自定义首页

```
- name: 写入首页（显示各自主机名）
  copy:
    content: "<h1>Hello from {{ inventory_hostname }}</h1>"
    dest: /var/www/html/index.html
    owner: nginx
    group: nginx
    mode: '0644'
```

- **模块**：`copy`，文件批量分发模块，这里通过 `content` 直接生成文件内容
- **核心变量**：`{{ inventory_hostname }}`
    - Ansible 内置变量，执行时自动替换为当前目标节点的别名（node1/node2/node3）
    - 效果：每台服务器的首页显示自己的主机名，方便验证负载均衡、快速区分节点
- **幂等性**：✅ 天然幂等。文件内容已一致则不修改，返回 `changed: false`。

### 任务 4：启动 Nginx 服务

```
- name: 确保 nginx 正在运行
  command: nginx
  args:
    creates: /run/nginx/nginx.pid
```

- **模块**：`command`，执行 Shell 命令，本身**不具备天然幂等性**
- **设计逻辑**：通过 `creates` 参数**模拟幂等**
    - `creates` 是 command 模块的特殊参数：**如果指定的文件已存在，就跳过命令不执行**
    - 这里通过判断 `/run/nginx/nginx.pid` 进程 pid 文件，间接判断 Nginx 是否已启动
- **本质**：用「文件是否存在」间接判断服务状态，是精简容器环境下的妥协写法。
- **幂等性**：⚠️ 伪幂等。仅靠文件判断，pid 残留时会判断失误，可靠性弱于官方专用模块。

---

## 四、写法的设计考量与适用场景

### 为什么这么写？

1. **适配精简容器环境**：多数 Alpine 极简镜像没有 `systemd` / `openrc` 服务管理器，无法使用 `service` 模块，只能直接执行二进制文件启动。
2. **最小化依赖**：不用额外安装服务管理组件，容器体积更小，实验环境搭建更快。
3. **基础幂等保证**：通过 `creates` 参数避免重复启动报错，比直接裸写 `command: nginx` 更稳健。

### 局限性

这是「实验环境能用」的写法，**不属于生产级规范写法**，仅适合快速搭建测试环境。

---

## 五、存在的问题与不规范之处

1. **服务状态判断不可靠** 仅靠 pid 文件判断服务是否运行，异常退出导致 pid 文件残留时，会出现「进程已经挂了，但脚本以为还在运行」的误判，不会自动拉起。
2. **无开机自启能力** 容器/服务器重启后，Nginx 不会自动启动，需要手动重新执行 Playbook。完整环境应配合服务管理模块设置开机自启。
3. **无法管理服务生命周期** 只能启动，不能停止、重启、重载配置；后续更新站点内容后，无法优雅重启 Nginx 加载新配置。
4. **路径兼容性问题** Alpine apk 安装的 Nginx 默认根目录是 `/var/lib/nginx/html`（部分旧版本或第三方镜像为 `/usr/share/nginx/html`），和脚本里的 `/var/www/html` 不匹配，页面放错位置会出现 404。
5. **缺少结果验证** 启动后没有校验端口监听、页面可访问性，无法确认服务真的正常运行。

---

## 六、优化后的规范版本（适配 Alpine 容器）

### 优化点

- 修正默认站点路径，适配 Alpine Nginx
- 增加服务健康校验步骤
- 优化启动逻辑，增加异常重启能力
- 保留实验环境兼容性，不依赖 systemd

```
---
- name: 优化版：部署 Nginx Web 服务
  hosts: web
  become: true
  tasks:

    - name: 1. 安装 Nginx
      apk:
        name: nginx
        state: present
        update_cache: yes

    - name: 2. 部署自定义首页
      copy:
        content: "<h1>Hello from {{ inventory_hostname }}</h1>"
        dest: /usr/share/nginx/html/index.html
        mode: '0644'

    - name: 3. 检查 Nginx 进程是否存活
      shell: "pgrep nginx || echo 'not running'"
      register: nginx_status
      changed_when: false

    - name: 4. 启动 Nginx（未运行时才执行）
      command: nginx
      when: nginx_status.stdout == 'not running'

    - name: 5. 验证服务可用性
      uri:
        url: http://127.0.0.1
        return_content: yes
        status_code: 200
      register: web_check
      changed_when: false
```

> 优化说明：用 `pgrep` 真实判断进程状态替代 pid 文件判断，准确性大幅提升；增加最终 HTTP 验证步骤，确保服务真正可用。