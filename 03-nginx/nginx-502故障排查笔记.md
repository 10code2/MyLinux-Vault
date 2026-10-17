

> 场景：WSL 2 Debian + nginx 1.26.3 反向代理实验
> 日期：2026-10-08
> 本笔记基于一次亲手经历的 502 故障：改错 proxy_pass 端口 → 502 → 看日志定位 → 修复 → 恢复 200

---

## 一、一次真实的 502 经历（本次实验全记录）

### 我做了什么
在站点配置 `mysite` 里，把反向代理的目标端口**故意改错**：

```nginx
location /api/ {
    proxy_pass http://localhost:9001/;   # ← 9000 改成 9001（9001 没有服务）
}
```

### 发生了什么
```
改 → nginx -t（语法通过！语法没错，nginx 不拦）→ reload → curl 验证

curl -I http://localhost:8080/api/
→ HTTP/1.1 502 Bad Gateway
```

### 怎么定位的
看错误日志（502 排查最关键的一步）：
```bash
tail -5 /var/log/nginx/mysite_error.log
```
```
connect() failed (111: Connection refused) while connecting to upstream
```
> 翻译：连接后端被拒绝——9001 端口根本没东西在监听。

### 怎么修复的
把端口改回 9000 → `nginx -t` → `service nginx reload` → curl 恢复 **200**。

**关键教训：`nginx -t` 只检查语法，不检查"转发目标是否活着"。** 语法对 ≠ 配置能工作。

---

## 二、502 是什么

```
502 Bad Gateway = nginx 已经把请求转发出去了，但目标后端连不上/没响应
```

**换句话说：反代规则生效了，但"下游"出了问题。**

这也说明：遇到 502，问题**不在 nginx 本身**（nginx 工作正常，是它"下游"的后端出了问题）。

---

## 三、状态码对照（别搞混）

| 状态码 | 含义 | 谁的问题 | 反代生效了吗 |
|---|---|---|---|
| **404** | 文件不存在 | nginx 自己找不到内容 | ❌ 规则没匹配 / 文件缺失 |
| **502** | 后端不可达 | **后端**（挂了、端口错、防火墙） | ✅ 转发了但目标无响应 |
| **504** | 后端超时 | 后端活着但**响应太慢**（超过超时时间） | ✅ 连上了但等不到 |

> 记忆口诀：404 是"我家没这道菜"，502 是"我点菜的后厨没人接电话"，504 是"后厨接了但一直说稍等"。

---

## 四、502 排查四步法（标准动作，背下来）

```
① 确认：  curl -I http://localhost:8080/api/        → 看到 502
② 日志：  tail /var/log/nginx/mysite_error.log      → 看具体报错（最有用的一步）
③ 测后端：curl http://localhost:9000/               → 后端活着吗？
④ 对症下药：
   - 后端挂了          → 启动后端服务
   - 端口写错          → 改 proxy_pass 端口
   - 后端只监听本机    → 检查后端 bind 地址/防火墙
```

**原则：永远先看日志，再猜原因。** 错误日志会直接告诉你"连接被拒绝"还是"连接超时"还是"找不到主机"，比任何猜测都快。

---

## 五、生产常见 502 原因清单

| 原因 | 典型表现 | 解决 |
|---|---|---|
| 后端服务挂了/没启动 | 日志：Connection refused | 启动/重启后端 |
| proxy_pass 端口写错 | 日志：Connection refused | 核对端口 |
| 后端只监听 127.0.0.1 | nginx 连不上（或反向） | 检查后端 bind 地址 |
| 防火墙拦截 | 日志：Connection timed out | 放行端口/安全组 |
| 后端启动慢 | 日志：upstream timed out | 增大 proxy_connect_timeout |
| upstream 组里后端全挂 | 日志：no live upstreams | 检查各后端健康状态 |

---

## 六、防患于未然（好习惯）

1. **改配置必测**：`nginx -t`（但记住它不测后端存活！）
2. **改配置用 reload 不用 restart**（不闪断）
3. **日志常看**：`tail -f /var/log/nginx/mysite_error.log`
4. **后端健康检查**：生产上用 nginx 的 `health_check`（商业版）或监控工具定时 curl 后端

---

## 七、反代配置速查（关联复习）

```nginx
location /api/ {
    proxy_pass http://localhost:9000/;              # 转发目标（结尾斜杠=剥离前缀）
    proxy_set_header Host $host;                    # 传真实域名
    proxy_set_header X-Real-IP $remote_addr;        # 传真实客户端 IP
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;  # 转发链路
    proxy_connect_timeout 5s;                       # 连后端超时（超时=504）
    proxy_read_timeout   30s;                       # 读后端响应超时
}
```

**502/504 和这两行超时设置直接相关**：后端响应超过 `proxy_read_timeout` → 504。

---

## 八、一句话总结

```
502 = nginx 活着，后端没接住。
排查顺序：curl 确认 → 看错误日志 → 测后端 → 修。
```
