# nginx 使用排障复盘（WSL 2 环境）

> 环境：Windows 10 专业工作站版 19045・WSL 3.0.1.0（MSI 版）・Debian 13 (trixie)・nginx 1.26.3
> 日期：2026-10-07
> 背景：WSL 2 中部署 nginx 学习运维，过程中连续遇到 4 个问题，全部解决。



***

## 问题一：网页打不开 —— WSL 空闲自动停止，nginx 随虚拟机消失

### 现象



* 部署完成后验证 HTTP 200，但稍后浏览器访问 `http://localhost:8080/` 打不开

* 进入 WSL 执行 `ps -ef | grep nginx`，只剩 grep 进程本身

* WSL 内 `curl localhost:8080` 返回 HTTP 000

### 排查过程



| 步骤        | 命令                            | 结果                 | 结论          |
| --------- | ----------------------------- | ------------------ | ----------- |
| 1. 看发行版状态 | `wsl -l -v`                   | **Debian Stopped** | WSL 虚拟机没在运行 |
| 2. 查进程    | `ps -ef \| grep nginx`        | 无 nginx 进程         | 服务确实没跑      |
| 3. 查端口    | `ss -tln \| grep 8080`        | 无监听                | 与步骤 2 印证    |
| 4. 交叉验证   | Windows `curl localhost:8080` | HTTP 200（恢复后）      | 转发链路正常      |

> ⚠️ 教训：过程中 
>
> `Test-NetConnection localhost -Port 8080`
>
>  曾误报 True（TCP 层探测），但实际 HTTP 请求失败 ——
>
> **以真实 HTTP 请求为最终验证标准**
>
> ，TCP 通 ≠ 服务可用。

### 根因

**新版 WSL 2 的 "空闲自动停止" 机制**：虚拟机空闲默认约 60 秒（`vmIdleTimeout=60000ms`）自动关闭，Linux 内所有进程（含 nginx）随之终止。WSL 虚拟机不运行时，Windows 到 WSL 的 [localhost](https://localhost) 端口转发也不工作。

### 解决方案

修改 `C:\Users\Administrator\.wslconfig`：



```
[wsl2]
vmIdleTimeout=-1        # 禁止空闲自动停止，服务常驻
```

### 验证



* `wsl -l -v` 显示 Debian **Running** 且不再自动停止

* Windows 侧 `curl localhost:8080` → HTTP 200，1415 bytes



***

## 问题二：执行 wsl 时反复弹 "远程桌面" 警告

### 现象

执行任何 wsl 命令时反复弹出两个警告：



1. 「无法加载远程桌面服务 ActiveX 控件，请确保 rdclientax.dll 在路径中」

2. 「RemoteApp 计算机无法连接到远程计算机」

### 排查过程



| 步骤           | 检查项                                               | 结果                          |
| ------------ | ------------------------------------------------- | --------------------------- |
| 1. 组件文件      | `System32\rdclientax.dll` / `SysWOW64` / `WinSxS` | **全部不存在**                   |
| 2. RDP 服务    | TermService / fDenyTSConnections                  | 已停止 / 已禁用（排除被入侵）            |
| 3. 远程会话      | 3 天 RDP 登录事件                                      | 无                           |
| 4. **时间线对比** | WSL 更新前 vs 更新后                                    | 更新前从不弹，**更新到 WSL 3.0 后开始弹** |

### 根因

**WSL 3.0 的 WSLg（Linux 图形界面支持）通过 RemoteApp / 远程桌面机制实现**。这台电脑是精简版系统，缺少 `rdclientax.dll`，每次启动 WSL 时 WSLg 尝试初始化远程桌面组件 → 加载失败 + 连接失败 → 反复弹窗。

> 定位关键：
>
> **时间线对比**
>
> ——"更新前没有、更新后出现" 把怀疑范围从 "系统被入侵" 收敛到 "WSL 更新引入的新组件（WSLg）"。

### 解决方案

在 `.wslconfig` 禁用 WSLg（纯命令行学习用不到 Linux GUI）：



```
[wsl2]
guiApplications=false   # 禁用 WSLg，不再触发远程桌面组件
```

### 验证

重启 WSL 后执行 wsl 命令不再弹窗。



***

## 问题三：启用 systemd 导致 WSL 虚拟机卡死

### 现象

为让 nginx 稳定运行，尝试启用 systemd：



* `.wslconfig` 加 `systemd=true` → wsl 报 "配置第 4 行错误"（解析警告）

* 改 `/etc/wsl.conf` 加 `[boot] systemd=true` → PID 1 变为 systemd，但 `systemctl is-system-running` 一直 **initializing**

* 随后所有 WSL 命令挂起，连 TaskStop 都超时 ——**虚拟机僵死**

### 排查过程



| 步骤        | 检查项                                 | 结果                                                    |
| --------- | ----------------------------------- | ----------------------------------------------------- |
| 1. 配置文件解析 | `.wslconfig` 报错定位                   | `systemd=true` 行解析异常                                  |
| 2. 换发行级配置 | `/etc/wsl.conf [boot] systemd=true` | systemd 成为 PID 1 但初始化不完成                              |
| 3. 系统日志提示 | 输出                                  | `Failed to start the systemd user session for 'root'` |

### 根因

**精简版系统缺少 systemd 集成所需的组件**，systemd 初始化无法完成（用户会话启动失败），导致 VM 僵死。这台机器跑不动 WSL 的 systemd 集成。

### 解决方案

**放弃 systemd 方案**，改用更轻量的启动命令机制（修改 `/etc/wsl.conf`）：



```
[boot]
command = service nginx start
```

每次 WSL 虚拟机启动时自动执行 `service nginx start`，效果等同自启，但不依赖 systemd。

### 验证



* 重启 WSL 后 nginx **master 进程自动运行**（父进程为 PID 1）

* `curl localhost:8080` → HTTP 200

> 思路：
>
> **遇到跑不通的复杂方案，不硬刚，换一条更轻的路径达成同一目标**
>
> 。



***

## 问题四（前置事件）：WSL 更新后发行版 "BasePath 丢失" 报错

### 现象

`wsl --update` 升级到 WSL 3.0.1.0 后，`wsl -l -v` 报错：



```
该注册的发行版 BasePath 丢失，注册配置无效 {bdf3c491-5c53-4fef-b469-a3d71aaf0dd4}
错误代码: Wsl/Service/RegisterDistro/ReadDistroConfig/E_UNEXPECTED
```

### 排查过程



| 步骤      | 检查项                                                   | 结果                                   |
| ------- | ----------------------------------------------------- | ------------------------------------ |
| 1. 数据文件 | 搜索 ext4.vhdx                                          | Ubuntu 1.3GB / Debian 0.6GB **完好**   |
| 2. 注册配置 | `HKCU\Software\Microsoft\Windows\CurrentVersion\Lxss` | Ubuntu/Debian 正常键 + 一个**空壳键**（无任何属性） |
| 3. 定位   | 错误 GUID 与空壳键比对                                        | 完全一致                                 |

### 根因

WSL 升级过程中产生了一个**损坏的空壳注册键**（无 BasePath 属性），WSL 服务枚举发行版时读取到它即报错。

### 解决方案

确认该键为空壳、不影响任何数据后，经用户确认删除：



```
Remove-Item 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Lxss\{bdf3c491-...}' -Recurse -Force
```

### 验证

`wsl -l -v` 恢复正常列出 Debian / Ubuntu，发行版数据完好。



***

## 总结：运维排查思路（本次沉淀的可复用方法论）

### 1. 症状分层排查（从用户感知到根因逐层下钻）



```
网页打不开（用户感知）
   ↓ 进程层：ps -ef | grep nginx        → 进程在吗？
   ↓ 端口层：ss -tln | grep 8080        → 监听了吗？
   ↓ 配置层：nginx -t                    → 语法对吗？
   ↓ 网络层：curl（WSL内 vs Windows侧）  → 转发/防火墙通吗？
```

### 2. 三板斧（nginx 排障永远先跑这三条）



| 命令                     | 回答的问题  |
| ---------------------- | ------ |
| `ps -ef \| grep nginx` | 进程是否存在 |
| `ss -tlnp \| grep 80`  | 端口是否监听 |
| `nginx -t`             | 配置是否合法 |

### 3. 分层验证：WSL 内 vs Windows 侧



* WSL 内 `curl localhost:8080` 通 = 服务层正常

* Windows 侧 `curl localhost:8080` 通 = 转发层正常

* 两者对比可快速隔离故障在 "服务" 还是 "转发 / 网络"

### 4. 时间线对比定位

"更新前正常、更新后异常" → 怀疑对象收敛到变更引入的东西（本次定位到 WSLg 就是这么找到的）。

### 5. WSL 2 环境的特殊性（不是 Linux 服务器）



| 特性                                | 坑               | 对策                             |
| --------------------------------- | --------------- | ------------------------------ |
| 虚拟机                               | 服务不随 Windows 保活 | `[boot] command` 自动拉起          |
| 空闲自动停止                            | 60 秒不用就停        | `vmIdleTimeout=-1`             |
| systemd 可选                        | 精简系统可能跑不动       | 优先轻量方案（service + boot command） |
| 配置文件解析                            | 中文注释会解析报错       | 只用 ASCII                       |
| [localhost](https://localhost) 转发 | 依赖虚拟机运行         | 先确认 `wsl -l -v` 是 Running      |

### 6. 修改原则



* **一次只改一个变量**，改完立即验证，再改下一个

* **系统级修改先说明影响、可回滚**（注册表删除前先确认空壳无数据）

* 验证以**真实请求结果**为准（HTTP 200），不信中间层探测的 "假阳性"



***

## 附录：最终生效配置

`C:\Users\Administrator\.wslconfig`



```
[wsl2]
guiApplications=false   # 禁用 WSLg，消除远程桌面弹窗
vmIdleTimeout=-1        # 禁止空闲自动停止，nginx 常驻
```

`/etc/wsl.conf`**（Debian 内）**



```
[boot]
command = service nginx start   # WSL 启动时自动拉起 nginx
```

**日常命令速查**



```
wsl -l -v                          # 查看发行版状态
wsl -d Debian                      # 进入 Debian
service nginx status               # 查看 nginx 状态
service nginx reload               # 重载配置（改配置后）
nginx -t                           # 测试配置语法
tail -f /var/log/nginx/mysite_access.log   # 实时查看访问日志
```