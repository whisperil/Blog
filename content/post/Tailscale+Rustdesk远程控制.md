---
title: "利用Tailscale和Rustdesk进行远程桌面控制"
date: 2026-07-23T16:00:00+08:00
draft: false   <-- 必须是 false
tags: ["Tailscale", "Rustdesk", "远程控制"]
---

利用Tailscale和Rustdesk进行远程桌面控制
**核心技术：P2P 直连（打洞）**
Tailscale 基于 WireGuard® 协议，它的核心强项是 **NAT 穿透（打洞）**。
## 第一部分：环境准备

### 1. 核心架构

- **电脑 A (Ubuntu - 公司)**：作为 **ID/中继服务器**，提供连接调度。
    
- **电脑 B & C (Windows)**：作为 **控制端/被控端**，通过虚拟网道互联。
    

### 2. 必备物料

- **Tailscale**: 用于构建跨地域的虚拟局域网。
    
- **RustDesk Server**: 包含 `hbbs` (ID服务) 和 `hbbr` (中继服务)。
    
- **RustDesk Client**: 在 Windows 和 Ubuntu 上安装的图形化控制软件。
    

---

## 第二部分：部署步骤

### 步骤 1：搭建 Tailscale 虚拟网道

在 **所有电脑** (A, B, C) 上安装并登录同一个账号：

1. **Ubuntu (A)**:
    
    - 执行 `curl -fsSL https://tailscale.com/install.sh | sh` 或使用静态包安装。
        
    - 执行 `sudo tailscale up` 并完成浏览器授权。
        
2. **Windows (B/C)**: 下载安装包登录。
    
3. **获取 IP**: 执行 `tailscale ip -4`，记下 Ubuntu (A) 的 IP（下文简称为 **`A-IP`**，如 `100.20.30.40`）。
    

### 步骤 2：在 Ubuntu (A) 部署 RustDesk 服务端

1. **准备目录**:
    
    Bash
    
    ```
    mkdir ~/rustdesk && cd ~/rustdesk
    # 将下载好的 rustdesk-server-linux-x64.zip 放入此目录并解压
    unzip rustdesk-server-linux-x64.zip
    chmod +x hbbs hbbr
    ```
    
2. **获取加密 Key**:
    
    - 首次运行 `./hbbs` 会在目录下生成 `id_ed25519.pub`。
        
    - 执行 `cat id_ed25519.pub`，**完整复制**这串长字符（下文简称为 **`A-Key`**）。
        

### 步骤 3：配置 Linux 系统自启动 (Systemd)

为了确保重启后依然可用，必须创建服务：

1. **创建 hbbs.service**: `sudo nano /etc/systemd/system/hbbs.service`
    
    Ini, TOML
    
    ```
    [Unit]
    Description=RustDesk ID Server
    After=network.target tailscaled.service
    
    [Service]
    Type=simple
    ExecStart=/home/你的用户名/rustdesk/hbbs -r A-IP:21117
    WorkingDirectory=/home/你的用户名/rustdesk/
    Restart=always
    
    [Install]
    WantedBy=multi-user.target
    ```
    
2. **创建 hbbr.service**: `sudo nano /etc/systemd/system/hbbr.service`
    
    _配置同上，只需将 `ExecStart` 改为 `/home/你的用户名/rustdesk/hbbr`。_
    
3. **启动服务**:
    
    Bash
    
    ```
    sudo systemctl daemon-reload
    sudo systemctl enable --now hbbs hbbr
    ```
    

### 步骤 4：配置 Windows 客户端 (B & C)

在 B 和 C 的 RustDesk 软件中进行相同配置：

1. **进入设置**: 设置 -> 网络 -> 解锁网络设置。
    
2. **ID 服务器**: 输入 **`A-IP`**。
    
3. **中继服务器**: 输入 **`A-IP`**。
    
4. **Key**: 粘贴 **`A-Key`**（确保无空格）。
    
5. **检查**: 看到主界面左下角显示 **“Ready (就绪)”**。
    

---

## 第三部分：常见问题与排错 (FAQ)

|**现象**|**原因**|**对策**|
|---|---|---|
|**显示 "Not Ready"**|Tailscale 断连或防火墙拦截|检查 `tailscale status`；Ubuntu 执行 `sudo ufw allow 21115:21119/tcp`|
|**显示 "Key 不匹配"**|复制 Key 时多出空格或 Key 已更新|重新 `cat id_ed25519.pub` 并清空 Windows 端的 Key 后重新粘贴|
|**服务状态 "Failed"**|路径写错或权限不足|使用 `pwd` 获取绝对路径填入 Service 文件；执行 `chmod +x`|
|**远程桌面黑屏**|Ubuntu 处于无头模式或 Wayland|物理机插虚拟显示器；登录界面切换至 **X11 (Xorg)**|

---

## 第四部分：安全建议

1. **固定密码**: 在被控端设置“永久密码”，方便无人值守连接。
    
2. **禁用公网**: 由于我们走 Tailscale 内部通道，建议在 Ubuntu 防火墙中仅允许 `tailscale0` 网卡的 21115-21117 端口访问，彻底杜绝外网扫描。
