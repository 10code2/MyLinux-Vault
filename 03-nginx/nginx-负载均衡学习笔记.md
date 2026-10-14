# nginx 负载均衡学习笔记（upstream）

> 环境：WSL 2 Debian + nginx 1.26.3 · 三个 python 后端（9000/9001/9002）
> 日期：2026-10-08
> 前置知识：反向代理（proxy_pass 转发到单个后端）

---

## 一、概念（What & Why）

### upstream 是什么
**upstream = 后端服务器组（名单）**。nginx 把请求轮流发给名单里的每一台。

```
浏览器 → nginx → ① 后端 9000
                ② 后端 9001   ← 轮流分发（默认 round-robin 轮询）
                ③ 后端 9002
```

### 为什么需要它（生产核心动机）

| 问题 | 解决 |
|---|---|
| 一台后端扛不住大量请求（性能瓶颈） | 多台分摊流量 |
| 一台后端挂了 → 网站全挂（单点故障） | 一台挂了，nginx 自动只发活着的 |

> 类比：一个窗口排长队 → 多开几个窗口。**大网站永远有多台服务器。**

---

## 二、配置（Where & How）

### upstream 块放在哪：server **外面**（http 级）

名单属于全局（所有站点共用），不属于任何单个站点：
```
upstream backend_pool { ... }   ← 放这：http 级（server 之外）
server { ... }                  ← 单个站点的规则
```

### 语法（重要：主机+端口缺一不可）

```nginx
upstream backend_pool {
    server 127.0.0.1:9000;   # 主机:端口 —— 只写 :9002 会报 "no host" 错误
    server 127.0.0.1:9001;
    server 127.0.0.1:9002;
}
```

### proxy_pass 的变化（新旧对比）

| 之前（单后端） | 现在（负载均衡） |
|---|---|
| `proxy_pass http://localhost:9000/;` | `proxy_pass http://backend_pool/;` |
| 具体地址：固定发 9000 | **名单名字**：nginx 去名单里轮流挑 |

### 完整配置示例

```nginx
upstream backend_pool {          # server 外，http 级
    server 127.0.0.1:9000;
    server 127.0.0.1:9001;
    server 127.0.0.1:9002;
}

server {
    ...
    location /api/ {
        proxy_pass http://backend_pool/;   # 指向组名，不再是具体端口
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### 部署四步口诀（不变）

```
改：记事本/vim 编辑 mysite
测：nginx -t
载：service nginx reload（或 nginx -s reload）
验：curl 访问看分布
```

---

## 三、验证方法（怎么确定轮询在工作）

**1 次请求只走其中一台**，必须连续访问多次、统计分布：

```bash
# 连续访问 12 次，统计各自收到几次
A=0; B=0
for i in $(seq 12); do
  R=$(curl -s http://localhost:8080/api/ | grep -o "BACKEND-[0-9]*")
  [ "$R" = "BACKEND-9000" ] && A=$((A+1)) || B=$((B+1))
done
echo "9000: $A 次, 9001: $B 次"
```

**预期**：两台/三台都有流量（如 6:6）。不是严格相等很正常——受响应时序、惩罚期记忆影响。

---

## 四、容错实验（高可用核心，最有价值）

**把一台后端杀掉，看 nginx 反应：**

```bash
# 找到 9001 的 PID（注意：不要用 pkill -f，会误杀自己）
PID=$(ss -tlnp | grep ":9001" | grep -oP "pid=\K[0-9]+" | head -1)
kill $PID
# 然后连续访问
for i in $(seq 6); do
  curl -s -o /dev/null -w "HTTP %{http_code} " http://localhost:8080/api/
  curl -s http://localhost:8080/api/ | grep -o "BACKEND-[0-9]*"
done
```

**结果对比（记住这个差异）：**

| 场景 | 后端挂了会怎样 |
|---|---|
| 单后端（直接 proxy_pass 某端口） | ❌ **502，全站挂** |
| 多后端（upstream 池） | ✅ **全 200，nginx 自动把请求发给活着的** |

**这就是负载均衡的本质：不是"分流量"，是"扛得住故障"。**

---

## 五、日志进阶（看每一单发给谁）

### 问题：默认访问日志看不到"发给了谁"

```
127.0.0.1 - - [08/Oct/2026:15:04:48] "GET /api/ HTTP/1.1" 200 70 "-" "curl/8.14.1"
```

只有请求和状态码，没有后端信息。

### 解决：自定义 log_format 加 $upstream_addr

```nginx
# 在 server 外（http 级）定义格式：
log_format upstream_log '$remote_addr -> $upstream_addr [$time_local] "$request" $status';

# server 里使用该格式：
access_log /var/log/nginx/mysite_access.log upstream_log;
```

**效果**：每行末尾出现 `-> 127.0.0.1:9000` —— **这一单发给了 9000**。

```
127.0.0.1 -> 127.0.0.1:9001 [08/Oct/2026:15:26:04] "GET /api/ HTTP/1.1" 200
127.0.0.1 -> 127.0.0.1:9002 [08/Oct/2026:15:26:04] "GET /api/ HTTP/1.1" 200
```

### 错误日志读法（排障关键）

```
[error] *35 connect() failed (111: Connection refused) while connecting to upstream
        ... upstream: "http://127.0.0.1:9002/", host: "localhost:8080"
        └ 最重要的一行：nginx 尝试连接的是 9002 —— 但 9002 没有服务！
```

---

## 六、幽灵后端排障（生产铁律）

### 现象
upstream 名单里写了**不存在的后端**（如 9002 没起服务）：
- 轮询"轮到"它 → 尝试连接 → 失败 → 标记失败（惩罚期 10s 内跳过）
- 轮询节奏被打乱 → 流量分配忽多忽少 → **间歇性故障**
- 错误日志刷 `connect refused`

### 定位
不猜，看错误日志的 `upstream: "http://127.0.0.1:9002/"` 一行——nginx 在尝试谁，一目了然。

### ⚠️ 生产铁律

> **upstream 名单里只写真实存在的后端。**

### 修复
- 方案 A：删掉不存在的行，回到干净轮询
- 方案 B：把真实后端跑起来（"幽灵"变"真身"），三台正常轮询

---

## 七、惩罚机制与进阶参数

### fail_timeout（惩罚期，默认 10s）
后端失败后，nginx **10 秒内不再给它派活**；恢复后要等窗口过去才重新尝试。

### 常用参数

```nginx
upstream backend_pool {
    server 127.0.0.1:9000 weight=3;                          # 权重：承担 3 倍流量
    server 127.0.0.1:9001 max_fails=2 fail_timeout=30s;      # 连败2次后 30s 内不派活
    server 127.0.0.1:9002 backup;                            # 备用：前面都挂才启用
}
```

---

## 八、本实验环境速查

```bash
# 三个后端（/var/www/backend1/2/3，内容 BACKEND-9000/9001/9002）
nohup python3 -m http.server 9000 --directory /var/www/backend1 > /tmp/backend1.log 2>&1 &
nohup python3 -m http.server 9001 --directory /var/www/backend2 > /tmp/backend2.log 2>&1 &
nohup python3 -m http.server 9002 --directory /var/www/backend3 > /tmp/backend3.log 2>&1 &

# 注意：WSL 重启后这些后台进程会消失，需重新拉起

# 验证
ss -tln | grep -E ':9000|:9001|:9002'        # 监听状态
curl -s -o /dev/null -w "%{http_code}" http://localhost:9000/   # 直连测试
tail -8 /var/log/nginx/mysite_access.log      # 看分发（$upstream_addr）
tail -5 /var/log/nginx/mysite_error.log       # 看失败
```

---

## 九、常见坑（这次亲手踩过的）

| 坑 | 现象 | 解决 |
|---|---|---|
| `server :9002;` 缺 IP | nginx -t 报 `no host in upstream` | 必须写 `server 127.0.0.1:9002;` |
| `nginx -reload` | 报 `invalid option: "r"` | nginx 没有 -reload；用 `nginx -s reload` 或 `service nginx reload` |
| `pkill -f "http.server 9001"` | **把自己也杀了**（命令行包含同样字符串） | 用 `ss -tlnp` 找 PID 再 `kill` |
| vim 里 `$变量` 看不见 | 语法高亮颜色与蓝色背景接近 | `:hi Identifier ctermfg=cyan` 或换配色 |
| 命令报错不会查 | 瞎猜 | 用 `nginx -h` / `命令 --help` 问它自己 |

---

## 十、一句话总结

```
upstream = 后端名单，轮流分发。
价值 = 分流量 + 扛故障（一台挂了网站不死）。
验证 = 日志里的 $upstream_addr（每单发给谁）。
铁律 = 名单只写真后端；故障看错误日志的 upstream 地址。
```
