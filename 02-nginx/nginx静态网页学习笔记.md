# nginx 静态网页与配置 学习笔记

> 环境：WSL 2（Debian 13）· nginx 1.26.3 · 本机 Windows 访问 http://localhost:8080
> 主题：静态网站托管 + nginx 配置入门（含排障经验）

---

## 一、核心认知（先把概念立住）

### 1. 网站 = 文件夹 + 服务器
把文件放进服务器指定的文件夹，就能通过网址访问：

| 你的操作 | 结果 |
|---|---|
| 放 `index.html` 到网站目录 | 访问 `/` 看到它 |
| 放 `photo.jpg` 到网站目录 | 访问 `/photo.jpg` 看到它 |

### 2. 数据与规则分离（最容易混的一点）
| 目录 | 放什么 | 用户能访问吗 |
|---|---|---|
| `/var/www/mysite/` | **内容**（网页、图片） | ✅ 能（这就是网站） |
| `/etc/nginx/` | **规则**（怎么工作） | ❌ 不能（也不应该） |

### 3. HTTP 一句话
浏览器和服务器对话的"语言"。**响应头决定浏览器行为**：
- `Content-Type: text/plain` → 显示
- `Content-Type: application/octet-stream` → 下载
- `charset=utf-8` → 中文正常；缺 charset → 中文乱码

---

## 二、nginx 目录结构（Debian 系）

```
/etc/nginx/
├── nginx.conf              ← 主配置（全局规则）
├── mime.types              ← 文件类型表
├── sites-available/        ← 规则草稿区（所有站点配置写这里）
│   └── mysite              ← 我们的站点配置
└── sites-enabled/          ← 规则生效区（符号链接）
    └── mysite → ../sites-available/mysite

/var/www/
└── mysite/                 ← 网站内容（放网页文件）
```

**为什么分 available / enabled 两层**：
- `sites-available` = 菜谱库（所有规则都在，不启用不生效）
- `sites-enabled` = 正在上桌的菜（符号链接指向生效的规则）
- 启用/停用站点 = 建/删链接，不动原文件 → 安全可恢复

---

## 三、配置文件：在哪、怎么改

### 位置
```
/etc/nginx/sites-available/mysite   ← 真正的配置（改这个）
/etc/nginx/sites-enabled/mysite     ← 链接（不用动）
```

### 两种编辑方法
**方法 A（新手友好）：Windows 记事本**
1. 文件管理器地址栏输入 `\\wsl$\Debian\etc\nginx\sites-available` 回车
2. 右键 `mysite` → 打开方式 → 记事本 → 改 → Ctrl+S 保存

**方法 B：WSL 命令行 nano**
```bash
wsl
nano /etc/nginx/sites-available/mysite
# 修改 → Ctrl+O 保存（回车）→ Ctrl+X 退出
```

### ⭐ 四步口诀（改配置的肌肉记忆）
```
改  →  测  →  载  →  验
改文件 → nginx -t（语法检查）→ reload（重载生效）→ 访问验证
```
```bash
nginx -t                  # 测试语法（必做！改错直接 reload 会让服务挂掉）
service nginx reload      # 重载生效
```

---

## 四、配置结构：层级与指令

### 层级（俄罗斯套娃）
```
http {                        ← 全局（所有站点）
    server {                  ← 一个站点
        location / {          ← URL 路径匹配（最细粒度）
            指令;             ← 具体规则
        }
    }
}
```

### 生产常见指令分类

| 分类 | 常见指令 | 作用 |
|---|---|---|
| 网络 | `listen` `server_name` | 端口、域名 |
| 路径 | `root` `index` `alias` | 根目录、默认首页 |
| 代理 | `proxy_pass` `proxy_set_header` | 反向代理 |
| 负载均衡 | `upstream` | 后端服务器组 |
| 安全 | `deny/allow` `auth_basic` `limit_req` `ssl_certificate` | IP封锁、密码、限流、HTTPS |
| 性能 | `gzip` `expires` `client_max_body_size` | 压缩、缓存、上传限制 |
| 跳转 | `return 301` `rewrite` | URL 跳转 |
| 日志 | `access_log` `error_log` | 记录 |
| 类型 | `default_type` `charset` `autoindex` | 文件类型、编码、目录列表 |

---

## 五、静态网页托管（已掌握技能）

### 1. 放文件 → 立即能访问
- 文件管理器 `\\wsl$\Debian\var\www\mysite` 直接拖文件进去
- 访问 `http://localhost:8080/文件名`

### 2. 改页面 → 不用 reload！
| 改的东西 | 需要的操作 |
|---|---|
| 网页文件（html/jpg） | ❌ 直接刷新浏览器 |
| nginx 配置 | ✅ `nginx -t` + `service nginx reload` |

### 3. 文件类型与编码（防止下载/乱码）
```nginx
charset utf-8;                    # 文本文件按 UTF-8 解析（防乱码）

location ~* \.md$ {               # .md 文件按纯文本返回（防下载）
    default_type text/plain;
}
```

### 4. 目录列表（autoindex）
```nginx
location / {
    autoindex on;              # 自动列出目录文件
    autoindex_exact_size off;  # 大小显示为 KB/MB
    autoindex_localtime on;    # 时间显示为本地时间
}
```
**优先级**：目录里有 `index.html` → 显示它（自定义首页）；没有 → 显示 autoindex 列表。

### 5. 自定义首页 = 更好的 autoindex
- 写一个漂亮的 `index.html` 放回目录，nginx 自动优先显示
- 代价：加新文件要手动更新链接
- 权衡：autoindex（自动但简陋）vs 自定义首页（好看但手动）

---

## 六、本次踩坑记录（都是宝贵经验）

| 问题 | 现象 | 根因 | 解决 |
|---|---|---|---|
| 扩展名重复 | 图片访问 404 | Windows 隐藏扩展名 → `1.png.png` | 重命名，开启"显示扩展名" |
| 文件下载 | .md 访问变下载 | nginx 不认 .md 类型 → 返回 octet-stream | `default_type text/plain` |
| 中文乱码 | .txt 中文乱码 | 响应头缺 charset，浏览器按 GBK 猜 | `charset utf-8` |
| types 覆盖坑 | 加 types 后 png/txt 全变下载 | `types` 指令会**替换**整个类型表 | 改用 `location + default_type` |
| 缓存假象 | 改配置后浏览器还是旧行为 | 浏览器缓存了旧响应 | **Ctrl+F5 硬刷新** |
| 中文文件名 | URL 一串 %E6%8E%92 编码 | 中文在 URL 里要编码 | 文件名用英文（行业惯例） |

**排障万能动作**：`curl -I http://localhost:8080/文件` → 看 HTTP 状态码和 Content-Type → 判断问题在哪。

---

## 七、常用命令速查

```bash
# 服务管理
service nginx start         # 启动
service nginx stop          # 停止
service nginx reload        # 重载配置（改配置后用，不断连接）
service nginx status        # 查看状态

# 配置
nginx -t                    # 测试配置语法
nginx -s reload             # 万能重载（不依赖 systemd/service）

# 排障（三板斧）
ps -ef | grep nginx         # 进程在吗
ss -tlnp | grep 8080        # 端口监听吗
tail -f /var/log/nginx/mysite_access.log   # 实时看日志

# 验证
curl -I http://localhost:8080/xxx   # 看响应头
curl -s http://localhost:8080/xxx   # 看响应内容
```

---

## 八、reload vs restart（重要区分）

| | reload | restart |
|---|---|---|
| 本质 | 热更新（不断连接） | 冷启动（先停再起，**连接全断**） |
| 适用 | **改配置（99% 场景）** | 升级版本、进程异常 |
| 铁律 | ✅ 日常用 | ❌ 生产环境改配置用它会被骂 |

**三种写法**（取决于环境）：
```bash
service nginx reload        # 无 systemd 环境（本机 WSL）
systemctl reload nginx      # 有 systemd 环境（云服务器）
nginx -s reload             # 万能（nginx 自带信号，处处可用）
```

---

## 九、下一步预告：生产核心技能

```
反向代理（已学）     proxy_pass → 单个后端
负载均衡（下一课）   upstream  → 一组后端轮询
HTTPS               ssl + return 301
限流/安全           limit_req / deny / auth_basic
```

**生产里 80% 时间就靠这几个规则**：`proxy_pass`、`upstream`、`ssl_certificate`、`limit_req`、`gzip/expires`。
