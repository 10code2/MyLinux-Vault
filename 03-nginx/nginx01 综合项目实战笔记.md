# nginx01 综合项目实战笔记

> 定位：把零散的 nginx 知识（静态托管/反向代理/负载均衡/HTTPS/安全）整合成完整站点，新增日志分析与故障演练
> 日期：2026-10-16 · 环境：WSL Debian + nginx 1.26.3
> 前置：nginx-静态网页 / nginx-负载均衡 / nginx-HTTPS / nginx-502故障排查

---

## 一、项目架构

```
用户（curl/浏览器）
   │
   ├─ HTTP :8080 ──→ 301 跳转 ──→ HTTPS :8443
   │
   └─ HTTPS :8443 ──→ nginx
                       ├─ /          → 静态站（/var/www/mysite，autoindex 目录列表）
                       ├─ /api/      → upstream 负载均衡 → 9000/9001/9002（python3 http.server）
                       ├─ /hello     → return 200 文本
                       ├─ /private   → IP白名单 + Basic Auth（.htpasswd）
                       ├─ 全局限流    → limit_req rate=1r/s burst=5 nodelay
                       └─ 访问日志    → 自定义格式（含 upstream_addr / request_time）
```

### 端口清单

| 端口 | 用途 |
|---|---|
| 8080 | HTTP 入口，301 跳转到 8443 |
| 8443 | HTTPS 入口（自签名证书） |
| 9000/9001/9002 | 后端服务（python3 -m http.server） |

---

## 二、完整配置（最终版）

文件：`/etc/nginx/sites-available/mysite`（软链到 sites-enabled）

```nginx
# ============================================
# 限速区：按客户端 IP 限流，1r/s + 突发5个不排队
# ============================================
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=1r/s;

# ============================================
# 自定义日志格式：含后端地址 + 后端响应时间 + 总耗时
# 注意：[$time_local] 后面必须有空格！否则字段错位
# ============================================
log_format main_ext '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    'upstream=$upstream_addr '
                    'upstream_time=$upstream_response_time '
                    'request_time=$request_time';

# ============================================
# 负载均衡：被动健康检查（30秒内失败3次→摘除30秒）
# ============================================
upstream backend_pool {
    server 127.0.0.1:9000 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:9001 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:9002 max_fails=3 fail_timeout=30s;
}

# ============================================
# HTTP → HTTPS 301 跳转
# ============================================
server {
    listen 8080;
    server_name localhost;
    return 301 https://$host:8443$request_uri;
}

# ============================================
# HTTPS 主站点
# ============================================
server {
    listen 8443 ssl;
    ssl_certificate     /etc/nginx/ssl/mysite.crt;
    ssl_certificate_key /etc/nginx/ssl/mysite.key;
    server_name localhost;
    root /var/www/mysite;
    index index.html;

    # 使用自定义日志格式
    access_log /var/log/nginx/mysite_access.log main_ext;
    error_log  /var/log/nginx/mysite_error.log;

    charset utf-8;

    # .md 文件按纯文本返回（Chrome 不识别 text/markdown）
    location ~* \.md$ {
        default_type text/plain;
    }

    # 静态站 + 目录索引
    location / {
        autoindex on;
        autoindex_exact_size off;
        autoindex_localtime on;
    }

    # 反向代理 + 负载均衡 + 限速
    location /api/ {
        proxy_pass http://backend_pool/;
        limit_req zone=api_limit burst=5 nodelay;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # 配置生效验证点
    location /hello {
        default_type text/plain;
        return 200 "Hello from nginx config!";
    }

    # IP白名单 + Basic Auth 双重保护
    location /private {
        allow 127.0.0.1;
        deny all;
        auth_basic "Restricted Area - Enter Password";
        auth_basic_user_file /etc/nginx/.htpasswd;
    }
}
```

### 后端启动脚本

```bash
mkdir -p /var/www/backend1 /var/www/backend2 /var/www/backend3
echo '<h1>BACKEND-9000</h1><p>served by python3 http.server on port 9000</p>' > /var/www/backend1/index.html
echo '<h1>BACKEND-9001</h1><p>served by python3 http.server on port 9001</p>' > /var/www/backend2/index.html
echo '<h1>BACKEND-9002</h1><p>served by python3 http.server on port 9002</p>' > /var/www/backend3/index.html

nohup python3 -m http.server 9000 --directory /var/www/backend1 > /tmp/backend1.log 2>&1 &
nohup python3 -m http.server 9001 --directory /var/www/backend2 > /tmp/backend2.log 2>&1 &
nohup python3 -m http.server 9002 --directory /var/www/backend3 > /tmp/backend3.log 2>&1 &
```

---

## 三、日志分析（核心新增能力）

### 3.1 自定义日志格式的 3 个关键字段

| 变量 | 含义 | 排障用途 |
|---|---|---|
| `$upstream_addr` | 实际打到的后端（如 `127.0.0.1:9001`） | 确认负载均衡是否均匀；502 时显示尝试过的所有后端（逗号分隔） |
| `$upstream_response_time` | 后端处理耗时（秒） | 找慢后端 |
| `$request_time` | nginx 收请求到发完响应的总耗时（秒） | 找慢请求 |

### 3.2 日志样例

```
127.0.0.1 - - [16/Oct/2026:19:20:21 +0800] "GET /api/ HTTP/1.1" 200 70 "-" "curl/8.14.1" upstream=127.0.0.1:9001 upstream_time=0.001 request_time=0.002
```

字段（空格分隔）：`$1`=IP，`$7`=路径，`$9`=状态码，`$13`=upstream，`$15`=request_time

### 3.3 五个 awk 分析命令

```bash
LOG=/var/log/nginx/mysite_access.log

# 1. PV / UV
echo "PV=$(wc -l < $LOG)"
echo "UV=$(awk '{print $1}' $LOG | sort -u | wc -l)"

# 2. 状态码分布
awk '{print $9}' $LOG | sort | uniq -c | sort -rn

# 3. 后端分布（负载均衡是否均匀）
awk '{print $13}' $LOG | sort | uniq -c | sort -rn

# 4. 慢请求 Top 10
awk '{
  for(i=1;i<=NF;i++) if($i ~ /^request_time=/) {split($i,a,"="); rt=a[2]}
  print rt, $7, $9
}' $LOG | sort -rn | head -10

# 5. URL 访问排行 Top 10
awk '{print $7}' $LOG | sort | uniq -c | sort -rn | head -10
```

### 3.4 新旧日志混合的过滤方法

改了 log_format 后，当前日志文件会混合旧格式（无 upstream= 字段）和新格式。旧格式字段数少，按固定列号取会错位。

**过滤方法**：只处理含 `request_time=` 的行
```bash
awk '/request_time=/{print $9}' $LOG | sort | uniq -c | sort -rn
```

**生产规范**：改 log_format 后 `> $LOG` 清空旧日志，或等 logrotate 轮转后新文件就是统一格式。

---

## 四、故障演练与排障（核心新增能力）

### 4.1 演练 1：单个后端挂了 → 无感知

```bash
pkill -f 'http.server 9001'
for i in $(seq 6); do curl -sk https://localhost:8443/api/; done
```

**现象**：6 次全成功，全是 9000/9002，9001 被静默跳过。

**原因**：`proxy_next_upstream error timeout`（默认开启）——连接后端失败时自动转发给下一个后端。

**日志验证**：`upstream` 字段里 9001 出现 0 次。

### 4.2 演练 2：所有后端挂了 → 502

```bash
pkill -f 'http.server 900'
curl -sk https://localhost:8443/api/
```

**现象**：返回 `502 Bad Gateway`。

**日志关键线索**：
```
upstream=127.0.0.1:9000, 127.0.0.1:9001, 127.0.0.1:9002
upstream_time=0.001, 0.000, 0.000
```
逗号分隔 = nginx 依次尝试了 3 个后端全部失败。

### 4.3 502 排障口诀

| upstream_addr 表现 | 含义 |
|---|---|
| 单个地址 | 单后端挂（但重试也失败或没开重试） |
| 多个地址逗号分隔 | nginx 依次尝试全部失败（全挂或网络不通） |
| `upstream=-` | 请求没经过反向代理（静态资源/认证拒绝） |

### 4.4 被动健康检查（生产必配）

nginx 开源版无主动健康检查，用 `max_fails` + `fail_timeout` 做被动摘除：

```nginx
upstream backend_pool {
    server 127.0.0.1:9000 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:9001 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:9002 max_fails=3 fail_timeout=30s;
}
```

**含义**：30 秒内失败 3 次 → 摘除该后端 30 秒（期间不转发）→ 30 秒后试探。

**为什么必配**：不配的话，一个后端挂了，每次轮到它都要等连接超时（默认 60s），拖慢整体响应。

### 4.5 503 vs 502 区分

| 状态码 | 原因 | 排查方向 |
|---|---|---|
| 502 Bad Gateway | nginx 连不上后端（后端挂了/端口错了/网络不通） | 看 upstream_addr，ss -tlnp 查后端 |
| 503 Service Unavailable | 被 limit_req 限流拒绝 | 看是否触发 rate+burst，调大 rate 或 burst |

---

## 五、生产配置 Checklist

- [ ] `nginx -t` 通过后必须 `service nginx reload`，且 `ps -ef | grep nginx` 确认 worker 时间刷新
- [ ] log_format 中 `[$time_local]` 后必须有空格，否则字段错位
- [ ] 改完 log_format 后清空旧日志或等轮转
- [ ] upstream 配 `max_fails` + `fail_timeout` 被动健康检查
- [ ] 静态资源 `charset utf-8`，`.md` 设 `default_type text/plain`
- [ ] 反向代理传 `Host` / `X-Real-IP` / `X-Forwarded-For`
- [ ] 敏感路径配 `auth_basic` + IP 白名单
- [ ] 日志含 `$upstream_addr` 和 `$request_time`（排障必备）

---

## 六、踩坑记录

| 坑 | 现象 | 原因 | 修复 |
|---|---|---|---|
| 改了配置不生效 | 负载均衡全打 9000 | 只跑了 nginx -t，没 reload | `service nginx reload`，看 ps 确认 worker 刷新 |
| log_format 少空格 | awk $9 取到 body_bytes，$7 取到 HTTP/1.1" | `[$time_local]` 和 `"$request"` 之间无空格 | 加空格，清空旧日志重测 |
| 新旧日志混合 | 状态码分布出现 3126/70/HTTP/1.1" | 旧格式无 upstream 字段，列号错位 | awk 加 `/request_time=/` 过滤，或清空日志 |
| 单后端挂了无 502 | 预期 502 实际全 200 | proxy_next_upstream 默认自动重试下一个后端 | 正常现象；全挂才 502 |
| 快速请求出现 503 | 第 7 次起 503 | limit_req rate=1r/s + burst=5，第 7 次超配额 | 正常现象；调大 rate/burst |
| sites-enabled 垃圾文件 | 目录里有 curl/done 0 字节文件 | 之前误操作重定向创建 | rm 清理 |
