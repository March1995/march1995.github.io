---
title: Docker Desktop (WSL2) 局域网暴露 2375 端口
date: 2026-06-26 
keywords: Docker WSL2
categories: [docker]
---

# Docker Desktop (WSL2) 局域网暴露 2375 端口

## 问题背景

Docker Desktop 使用 WSL2 后端时，WSL2 虚拟机有独立的网络栈（NAT 模式），外部流量无法直接到达 Docker 守护进程。即使开启 Docker Desktop 的 "Expose daemon on tcp://localhost:2375" 选项，也仅绑定 `127.0.0.1`，局域网其他机器无法访问。

## 架构原理

```
局域网客户端
    │
    ▼
Windows 物理网卡 (11.168.2.30 / 192.168.192.3)
    │
    ├── ① Windows 防火墙 (仅允许指定网段)
    │
    ▼
netsh portproxy (端口转发)
    │
    ▼
127.0.0.1:2375
    │
    ▼
Docker Desktop → WSL2 虚拟机 → Docker Daemon
```

> **为什么需要 portproxy？**  
> WSL2 虚拟机 IP（如 `172.18.205.179`）处于 NAT 子网，局域网无法直接路由。必须通过 Windows 的端口转发将流量从物理网卡中转到 localhost，再由 Docker Desktop 代理到 WSL2 内部的 Docker 守护进程。

## 操作步骤

### 1. 清理 daemon.json（避免冲突）

WSL2 模式下，**不要在 `daemon.json` 中配置 `hosts`**，这会与 Docker Desktop 的守护进程管理冲突。

文件路径：`%USERPROFILE%\.docker\daemon.json`

```json
{
  "builder": {
    "gc": {
      "defaultKeepStorage": "20GB",
      "enabled": true
    }
  },
  "experimental": false,
  "insecure-registries": [
    "cr.registry.res.rdcentercloud.com",
    "cr.authentication.res.rdcentercloud.com"
  ],
  "registry-mirrors": [
    "https://docker.1panel.live"
  ]
}
```

> 确保文件中**没有** `"hosts"` 字段。

### 2. 启用 Docker Desktop TCP 暴露

修改文件：`%APPDATA%\Docker\settings-store.json`

```json
{
  "ExposeDockerAPIOnTCP2375": true
}
```

等价于 Docker Desktop GUI：Settings → General → 勾选 `Expose daemon on tcp://localhost:2375 without TLS`。

### 3. 配置 Windows 防火墙

以**管理员身份**运行 PowerShell：

```powershell
# 添加防火墙入站规则（限制来源网段）
netsh advfirewall firewall add rule name="Docker 2375" `
    dir=in action=allow protocol=TCP `
    localport=2375 `
    remoteip=11.168.2.0/24,192.168.192.0/24
```

| 参数 | 说明 |
|------|------|
| `remoteip` | **强烈建议**限制为局域网网段，避免暴露到公网 |
| `localport` | Docker API 端口 2375 |

> 如果已有规则需要追加网段，使用 `set`：
> ```powershell
> netsh advfirewall firewall set rule name="Docker 2375" `
>     new remoteip=11.168.2.0/24,192.168.192.0/24
> ```

### 4. 配置端口转发

```powershell
# 为每个本机 IP 添加转发规则
netsh interface portproxy add v4tov4 `
    listenport=2375 listenaddress=11.168.2.30 `
    connectport=2375 connectaddress=127.0.0.1

netsh interface portproxy add v4tov4 `
    listenport=2375 listenaddress=192.168.192.3 `
    connectport=2375 connectaddress=127.0.0.1
```

| 参数 | 说明 |
|------|------|
| `listenaddress` | 本机物理网卡 IP（替换为实际 IP） |
| `connectaddress` | 目标地址，固定为 `127.0.0.1` |

### 5. 重启 Docker Desktop

右键系统托盘 Docker 图标 → **Quit Docker Desktop** → 重新启动。

## 验证

### 本机检查

```powershell
# 查看端口监听
netstat -ano | findstr 2375

# 查看端口转发
netsh interface portproxy show v4tov4

# 查看防火墙规则
netsh advfirewall firewall show rule name="Docker 2375"
```

### 局域网测试

在其他机器上：

```bash
docker -H tcp://11.168.2.30:2375 info
docker -H tcp://11.168.2.30:2375 ps
```

## 安全提醒

| 风险 | 说明 |
|------|------|
| **无 TLS 加密** | Docker API 明文传输，局域网内可被嗅探 |
| **等同于 root** | 访问 Docker API 等于拥有宿主机 root 权限 |
| **必须限制来源** | 防火墙 `remoteip` 务必限制为信任网段 |

### 增强安全措施（可选）

**方案 A - 防火墙限制来源 IP：**
```powershell
netsh advfirewall firewall set rule name="Docker 2375" `
    new remoteip=11.168.2.0/24,192.168.192.0/24
```

**方案 B - 配置 TLS（生产环境推荐）：**
```json
// daemon.json
{
  "tls": true,
  "tlscacert": "C:\\certs\\ca.pem",
  "tlscert": "C:\\certs\\server-cert.pem",
  "tlskey": "C:\\certs\\server-key.pem"
}
```

客户端连接：
```bash
docker -H tcp://11.168.2.30:2376 --tlsverify \
    --tlscacert=ca.pem --tlscert=cert.pem --tlskey=key.pem info
```

## 故障排查

### 问题 1：端口转发不生效

```powershell
# 删除重建
netsh interface portproxy delete v4tov4 listenport=2375 listenaddress=11.168.2.30
netsh interface portproxy add v4tov4 listenport=2375 listenaddress=11.168.2.30 connectport=2375 connectaddress=127.0.0.1
```

### 问题 2：IP 转发未启用

```powershell
# Windows 默认需要启用 IP 转发
Get-NetIPInterface | Where-Object {$_.InterfaceAlias -like "*WSL*"}
```

### 问题 3：Docker Desktop 不监听 2375

确认 `settings-store.json` 中 `ExposeDockerAPIOnTCP2375` 为 `true`，且 `daemon.json` 中**没有** `hosts` 字段。

### 问题 4：重启后端口转发丢失

`netsh interface portproxy` 规则是持久化的，重启后仍然保留。但如果发现丢失，重新添加即可。

## 环境信息

| 项 | 值 |
|------|------|
| OS | Windows 11 Pro |
| Docker Desktop | 27.3.1 (WSL2 后端) |
| 物理网卡 IP | 11.168.2.30, 192.168.192.3 |
| 防火墙允许网段 | 11.168.2.0/24, 192.168.192.0/24 |
