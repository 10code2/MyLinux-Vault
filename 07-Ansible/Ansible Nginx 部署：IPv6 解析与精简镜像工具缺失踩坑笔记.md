---
tags: [ansible, 踩坑, ipv6, alpine, 排障]
created: 2026-10-24
---
## 一、故障背景

> 相关笔记：[[Ansible Nginx 部署全链路踩坑运维笔记]]｜[[实操步骤]]

### 现象描述

Playbook 全量执行成功，`uri` 模块验证服务返回 200 正常；但手动执行 `wget localhost` 批量报 `Connection refused`，同时 `ss` 命令提示不存在，双重现象叠加极易误判为「Nginx 未启动、端口未监听」。

### 核心结论

**Nginx 服务本身完全正常，故障是两个环境特性叠加形成的假象**：

1. Alpine 精简镜像默认未预装网络工具，`ss` 命令不存在，无法验证端口状态
2. `localhost` 域名优先解析为 IPv6 地址，而 Nginx 默认仅监听 IPv4 端口，IPv6 端口无服务导致连接被拒

---

## 二、坑点 1：精简镜像缺失网络工具，误判端口状态

### 故障现象

执行端口查看命令时报错：

```
/bin/sh: ss: not found
```

结合连接拒绝的报错，很容易直接得出「80 端口未监听、Nginx 未启动」的错误结论。

### 根因分析

Alpine Linux 为极致压缩镜像体积，默认未安装 `iproute2` 工具集，`ss`、`ip` 等常用运维命令均未预装，属于镜像本身的特性，并非系统或服务故障。

### 替代验证方案

#### 方案 1：安装 net-tools 工具集

```
# 批量安装
ansible web -i inventory -m apk -a "name=net-tools state=present"

# 使用 netstat 查看端口
ansible web -i inventory -m command -a "netstat -tlnp | grep :80"
```

#### 方案 2：零依赖读取内核原生统计

无需安装任何工具，直接读取系统内核文件验证端口：

```
ansible web -i inventory -m shell -a "cat /proc/net/tcp | head -10"
```

### 知识点沉淀

1. 轻量发行版默认会裁剪大量运维工具，遇到 `command not found` 优先考虑工具缺失，不要直接判定服务异常。
2. 排障必须多维度交叉验证，不能仅凭单一命令的输出下结论。

---

## 三、坑点 2：localhost IPv6 解析优先级，造成连接拒绝假象

### 故障现象

同一台节点出现完全矛盾的测试结果：

- `wget localhost` → `Connection refused` 连接失败
- `wget 127.0.0.1` → 正常返回页面内容
- Playbook 中 `uri` 模块使用 `http://127.0.0.1` 验证 → 200 正常

### 根因深度拆解

1. **localhost 是域名不是固定 IP**：系统 `hosts` 文件默认将 localhost 同时解析到两个地址
    - IPv4 地址：`127.0.0.1`
    - IPv6 地址：`::1`
2. **解析优先级问题**：多数 Linux 发行版的 `wget`、`curl` 工具默认优先解析 IPv6 地址
3. **Nginx 默认监听行为**：配置仅写 `listen 80` 时，Nginx 默认仅监听 IPv4 的 80 端口，IPv6 的 80 端口无任何服务
4. **最终假象**：请求打到 `::1:80` 时被直接拒绝，表面看和「服务未启动」的报错完全一致，实际 IPv4 服务完全正常

### 验证方法

强制指定 IPv4 协议访问，正常则 100% 确认是 IPv6 解析导致的问题：

```
ansible web -i inventory -m command -a "wget -4 -qO- localhost"
```

### 修复方案

#### 推荐方案：Nginx 同时监听 IPv4 + IPv6（一劳永逸）

修改站点配置，显式增加 IPv6 监听指令，兼容所有访问方式：

```
- name: 写入统一站点配置
  copy:
    content: |
      server {
          listen 80 default_server;
          listen [::]:80 default_server;
          root /usr/share/nginx/html;
          index index.html;
      }
    dest: /etc/nginx/http.d/default.conf
  notify: 重载 Nginx 配置
```

#### 临时方案：命令强制指定 IPv4

仅适合临时测试，不推荐作为长期方案：

```
wget -4 -qO- localhost
curl -4 localhost
```

### 知识点沉淀

1. **`localhost ≠ 127.0.0.1`**：前者是域名，需要经过系统解析，可能对应多个地址；后者是固定的 IPv4 地址。
2. 出现「不同工具测试结果矛盾」时，优先对比测试条件差异（地址、协议、端口、路径），90% 的问题都源于测试条件不一致。
3. 生产环境服务建议同时兼容 IPv4/IPv6，避免解析优先级差异导致的访问异常。

---

## 四、故障复盘与排障思路

### 完整假象传导链路

```
ss 命令缺失 → 无法验证端口，误以为端口未监听
        +
wget localhost 走 IPv6 被拒 → 误以为服务未启动
        ↓
双重假象叠加 → 错误定性为 Nginx 进程异常
```

实际情况：Nginx 进程正常、IPv4 端口正常，仅存在「工具未安装」和「IPv6 未监听」两个外围问题。

### 排障核心原则

1. **先验工具，再信输出**：命令报错先确认命令本身是否存在，不要直接把输出等同于系统状态。
2. **矛盾结果控制变量**：出现不一致的测试结果时，固定其他条件，只改变一个参数（如把 localhost 换成 127.0.0.1），快速定位差异点。
3. **多维度交叉验证**：服务状态要从「进程存在性 → 端口监听 → 业务请求」三个维度交叉印证，单一维度不能定性。

---

## 五、避坑指南

1. **基础工具预装**：Alpine 实验环境提前预装常用运维工具（`iproute2`、`net-tools`、`curl`），避免排障时被工具缺失打断节奏。
2. **双栈监听默认配置**：写 Nginx 配置时默认同时监听 IPv4 + IPv6，兼容所有访问方式，杜绝解析优先级带来的隐性问题。
3. **测试用明确 IP**：排障验证优先使用明确的 IP 地址，避免域名解析、hosts 规则带来的额外干扰。
4. **拒绝单一现象定性**：任何故障都要至少两个维度交叉验证后再下结论，避免被表面假象带偏排查方向。