---
title: nginx-HTTPS学习笔记
created: 2026-10-08
updated: 2026-10-24
status: 待复习
tags: [nginx, web服务]
---

# nginx HTTPS 学习笔记

> 环境：WSL 2 Debian + nginx 1.26.3 · 自签名证书（本机学习用）
> 日期：2026-10-08
> 前置：静态托管 + 负载均衡已掌握

---

## 一、HTTPS 是什么（HTTP vs HTTPS）

```
HTTP  = 明文传输（明信片）：路上任何人都能看见内容
HTTPS = 加密传输（保险箱）：数据加密后再上路，只有收件人有钥匙
```

| | HTTP | HTTPS |
|---|---|---|
| 传输内容 | 明文 | 加密 |
| 浏览器标记 | "不安全" | 锁图标 |
| 适用 | 纯浏览场景 | **一切有敏感数据的场景** |

## 二、为什么必须 HTTPS（生产铁律）

| 风险 | 后果 |
|---|---|
| 明文传输 | 密码、个人信息可被抓包直接看到 |
| 中间人篡改 | 数据可被改（网页被注入广告/恶意代码） |
| 浏览器不信任 | 访问 HTTP 网站被标记"不安全" |

**铁律：只要有登录、支付、个人信息的网站，必须 HTTPS。**

## 三、证书机制（锁和钥匙）

HTTPS 依赖一对"锁和钥匙"：

| 东西 | 学名 | 作用 |
|---|---|---|
| 锁（公开） | **证书** certificate | 发给浏览器，用它加密 |
| 钥匙（保密） | **私钥** private key | 服务器自己留，用它解密 |

**握手过程（一句话版）**：
```
① 浏览器：我要访问
② nginx：这是我的证书（锁），给你
③ 浏览器：验证锁真伪 → 用锁加密一个"会话密码"发过去
④ nginx：用钥匙解开 → 双方用密码加密通信 ✅
```

## 四、自签名证书 vs 受信任证书

| | 自签名 | 受信任（CA 签发） |
|---|---|---|
| 谁签名 | 自己给自己签 | 权威机构（如 Let's Encrypt） |
| 类比 | 自制身份证自己盖章 | 公安局办的身份证 |
| 浏览器 | ❌ 警告"不可信" | ✅ 自动信任 |
| 用途 | 本机学习、内网测试 | 生产网站 |

**浏览器警告不是网站坏了**——是浏览器看到"自制身份证"，出于安全拒绝自动信任。点"高级→继续"是手动放行。

## 五、配置（Where & How）

### 文件位置
```
/etc/nginx/sites-available/mysite
（Windows 记事本：\\wsl$\Debian\etc\nginx\sites-available\mysite）
```

### 加在哪：server 块内、listen 8080; 正下方

**加之前：**
```nginx
server {
    listen 8080;
    server_name localhost;
```

**加之后（新增 4 行）：**
```nginx
server {
    listen 8080;
    listen 8443 ssl;                          # 新增 HTTPS 入口（8443，避开 443）
    ssl_certificate     /etc/nginx/ssl/mysite.crt;   # 锁的地址
    ssl_certificate_key /etc/nginx/ssl/mysite.key;   # 钥匙的地址
    server_name localhost;
```

**为什么加在这**：HTTPS 入口 = 一个监听端口，所以加在监听区（listen 旁边）；证书地址是配套，惯例紧跟。

### 部署四步口诀（不变）
```
改：加 4 行 → 测：nginx -t → 载：service nginx reload → 验：curl
```

### 验证
```bash
curl -sk https://localhost:8443/     # -k = 忽略证书验证（自签名）
# 不带 -k 会报证书错误 —— 和浏览器警告同理
```

## 六、HTTP→HTTPS 自动跳转（301，生产标配）

### 什么是 301
**301 = 永久重定向**：地址永久搬家，浏览器自动跳到新地址。

```
用户访问 http://localhost:8080/mynote.html
        ↓ nginx 返回 301 + Location
浏览器自动跳到 https://localhost:8443/mynote.html（路径原样保留）
```

### 为什么 80 端口必须跳 443
- 用户输网址很少主动打 `https://`，浏览器默认走 HTTP
- 不跳转 = 留了个"裸奔通道"，HTTPS 白装
- SEO：http 和 https 是两个站，301 把权重合并

### 配置：拆成两个 server（各管各的）

```nginx
# ===== server 1：HTTP 入口，一律踢到 HTTPS =====
server {
    listen 8080;
    server_name localhost;
    return 301 https://$host:8443$request_uri;
    # $host = 用户访问的域名，$request_uri = 请求的路径
}

# ===== server 2：HTTPS，真正站点 =====
server {
    listen 8443 ssl;
    ssl_certificate     /etc/nginx/ssl/mysite.crt;
    ssl_certificate_key /etc/nginx/ssl/mysite.key;
    server_name localhost;
    root /var/www/mysite;
    ... # 其余 location 规则不变
}
```

### 验证
```bash
curl -I http://localhost:8080/          # 301 + Location: https://localhost:8443/
curl -I http://localhost:8080/mynote.html   # 跳转后路径还在
```

### 301 vs 302（重要区别）

| | 301 永久 | 302 临时 |
|---|---|---|
| 含义 | 地址永久搬家 | 暂时换地方 |
| 浏览器 | 记住新地址 | 每次重新问 |
| SEO | 权重合并 | 不合并 |
| 用途 | HTTP→HTTPS | 维护页等临时场景 |

## 七、生产方案：Let's Encrypt（免费受信任证书）

```bash
apt install certbot python3-certbot-nginx
certbot --nginx -d example.com    # 自动申请证书 + 配置 + 跳转
```

| | 自签名（学习） | Let's Encrypt（生产） |
|---|---|---|
| 信任 | 浏览器警告 | 自动信任 |
| 获取 | openssl 一条命令 | certbot 一条命令 |
| 有效期 | 自定 | 90 天（自动续期） |

## 八、三个概念的澄清（协议 / 端口 / 路径）

```
协议 = 加不加密（http:// 明文 / https:// 加密）
端口 = 找谁（哪个服务在听）
路径 = 要什么（哪个资源/文件）

http://localhost:8080/mynote.html
│     │        │       └ 路径：要 mynote.html
│     │        └ 端口：8080（nginx 在听）
│     └ 主机
└ 协议：HTTP（明文）
```

**一个端口可挂无数资源**：`/`（首页）、`/mynote.html`（文件）、`/api/`（转发后端）……路径决定 nginx 送你去哪。

## 九、入口收敛（安全架构核心）

```
❌ 生产禁止：用户 → 9000 后端（绕过 nginx = 绕过 HTTPS/限流/防护/日志）
✅ 正确：用户 → 8080/8443（nginx 唯一入口）→ 9000/9001/9002（用户摸不到）
```

**端口不是网站专属**——任何程序都能监听（python 后端也听 9000）。生产里后端只监听内网/被防火墙挡死，**用户物理上够不着后端**，只能从 nginx 这个口进。

## 十、本实验命令速查

```bash
# 生成自签名证书（有效期 1 年）
mkdir -p /etc/nginx/ssl
cd /etc/nginx/ssl
openssl req -x509 -newkey rsa:2048 -keyout mysite.key -out mysite.crt -days 365 -nodes -subj "/CN=localhost"

# 验证
curl -sk https://localhost:8443/         # 页面内容
curl -skI https://localhost:8443/        # 响应头
# 浏览器体验：访问 https://localhost:8443/ → 看警告 → 高级→继续
```

## 十一、常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 自签名证书警告 | 浏览器"不是私密连接" | 正常！学习环境手动放行；生产用 certbot |
| `/etc/nginx/ssl` 不存在 | 生成证书找不到目录 | 先 `mkdir -p /etc/nginx/ssl` |
| 证书生成在别处 | nginx 找不到证书 | 证书/私钥要放在配置指定的路径 |
| 443 端口冲突 | 默认站点占着 443 | 学习环境用 8443 避开 |

## 十二、一句话总结

```
HTTPS = 证书（锁）+ 私钥（钥匙）加密通信。
配置只需 3 行：listen 8443 ssl + 证书路径两行。
HTTP 入口用 return 301 全部踢到 HTTPS（生产标配）。
学习用自签名（浏览器警告正常），生产用 certbot 申请受信任证书。
后端永远不能直接暴露给用户 —— 入口只留 nginx。
```

## 🔗 相关笔记

- [[nginx-负载均衡学习笔记]]
