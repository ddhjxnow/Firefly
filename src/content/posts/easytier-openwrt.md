---

title: openwrt安装easytier

published: 2026-02-04

description: openwrt安装easyteir教程

tags: [easyteir, openwrt]

draft: false

---
# OpenWrt 部署 EasyTier 实现内网穿透与组网教程

本教程介绍如何在 OpenWrt 路由器（以 RAX3000M aarch64 架构为例）上安装并配置 **EasyTier**，实现跨地域的内网互联。

## 1. 下载与安装

首先，前往 [GitHub Releases](https://github.com/EasyTier/EasyTier/releases) 确认适合你硬件架构的版本。

```bash
# 下载对应架构的压缩包（以 v2.4.5 aarch64 为例）
wget -O easytier.zip https://github.com/EasyTier/EasyTier/releases/download/v2.4.5/easytier-linux-aarch64-v2.4.5.zip

# 解压文件
unzip easytier.zip

# 将核心程序移动至系统路径并赋予执行权限
cp easytier-core easytier-cli /usr/bin/
chmod +x /usr/bin/easytier-core /usr/bin/easytier-cli

```

---

## 2. 命令行测试

在正式配置自启动前，建议先手动运行命令以确保各项参数配置正确。

```bash
/usr/bin/easytier-core \
  --hostname openwrt \
  --network-name <你的网络名称> \
  --network-secret <你的网络密码> \
  --private-mode true \
  --peers tcp://<公网服务器IP>:11010 \
  --listeners tcp://0.0.0.0:11010 \
  --dev-name easytier \
  -i <虚拟局域网IP，如192.168.10.3> \
  --proxy-networks <本网段LAN地址，如192.168.1.0/24>

```

**关键参数说明：**

* `--network-name`: 虚拟网络名称，用于识别身份。
* `--network-secret`: 网络加入密码。
* `--proxy-networks`: 允许远程节点访问你 OpenWrt 下的 LAN 网段。
* `--peers`: 用于辅助打洞的公网节点（或已知公网 IP 的对端节点）。

---

## 3. 配置 Procd 守护进程自启动

为了保证路由器重启后服务能自动运行，我们需要在 `/etc/init.d/` 下创建一个启动脚本。

### 创建脚本

执行 `vi /etc/init.d/easytier`，写入以下内容（请替换其中的占位符）：

```bash
#!/bin/sh /etc/rc.common

USE_PROCD=1
START=99

start_service() {
    procd_open_instance
    procd_set_param command /usr/bin/easytier-core \
        --hostname openwrt \
        --network-name <你的网络名称> \
        --network-secret <你的网络密码> \
        --private-mode true \
        --peers tcp://<公网服务器IP>:11010 \
        --listeners tcp://0.0.0.0:11010 \
        --dev-name easytier \
        -i <虚拟局域网IP> \
        --proxy-networks <本网段LAN地址>

    procd_set_param respawn  # 异常退出自动重启
    procd_set_param stdout 1 # 日志记录
    procd_set_param stderr 1
    procd_close_instance
}

```

### 启用服务

```bash
chmod +x /etc/init.d/easytier
/etc/init.d/easytier enable
/etc/init.d/easytier start

```

---

## 4. 配置防火墙（打通局域网互访）

这是最容易被忽略的一步。不配置防火墙，数据包会被 OpenWrt 拦截。

### 编辑配置文件

执行 `vi /etc/config/firewall`，在文件末尾添加：

```text
# 1. 定义 EasyTier 安全区域
config zone
    option name 'easytier'
    option input 'ACCEPT'
    option output 'ACCEPT'
    option forward 'ACCEPT'
    option device 'easytier'

# 2. 允许 LAN 到 EasyTier 的转发
config forwarding
    option src 'lan'
    option dest 'easytier'

# 3. 允许 EasyTier 到 LAN 的转发
config forwarding
    option src 'easytier'
    option dest 'lan'

```

### 生效配置

```bash
/etc/init.d/firewall reload

```

---

## 5. 验证状态

使用内置的 CLI 工具查看节点连接情况：

```bash
easytier-cli peer

```

如果能看到其他节点的 `Hostname` 和 `Virtual IP`，说明组网成功。

---

