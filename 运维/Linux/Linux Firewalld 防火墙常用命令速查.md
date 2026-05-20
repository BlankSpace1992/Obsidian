---
title: Linux Firewalld 防火墙常用命令速查
tags:
  - Linux
  - 防火墙
  - firewalld
  - 运维
  - 安全
---

# Linux Firewalld 防火墙常用命令速查

> `firewall` 不是一条具体命令，指 Linux 中由 **firewalld** 提供的动态防火墙管理功能，核心命令行工具是 `firewall-cmd`。
> CentOS/RHEL 7+ 及现代发行版默认使用，替代旧版 iptables。

---

## 服务管理（systemctl）

```bash
systemctl start firewalld      # 启动
systemctl stop firewalld       # 停止
systemctl enable firewalld     # 开机自启
systemctl disable firewalld    # 禁用自启
```

---

## 状态查看

```bash
firewall-cmd --state                   # 运行状态（running / not running）
systemctl status firewalld             # 服务状态（详细信息）
firewall-cmd --get-default-zone        # 查看默认区域
firewall-cmd --get-active-zones        # 查看活动区域及关联网卡
```

---

## 端口管理

### 开放端口

```bash
# 临时生效（重启后失效）
firewall-cmd --add-port=80/tcp

# 永久生效（写入配置，需 reload）
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --reload
```

### 关闭端口

```bash
firewall-cmd --permanent --remove-port=80/tcp
firewall-cmd --reload
```

### 查看已开放端口

```bash
firewall-cmd --list-ports
```

---

## 服务管理

### 开放服务

```bash
# 临时
firewall-cmd --add-service=http

# 永久
firewall-cmd --permanent --add-service=ssh
firewall-cmd --reload
```

### 关闭服务

```bash
firewall-cmd --permanent --remove-service=http
firewall-cmd --reload
```

### 查看支持的服务列表

```bash
firewall-cmd --get-services
```

---

## 规则查看

```bash
firewall-cmd --list-all                         # 默认区域的所有规则
firewall-cmd --zone=public --list-all           # 指定区域的所有规则
```

---

## 重新加载规则

```bash
firewall-cmd --reload            # 重新加载（不中断现有连接）
firewall-cmd --complete-reload   # 完全重新加载（会中断现有连接）
```

---

## 恐慌模式

```bash
firewall-cmd --panic-on      # 启用（拒绝所有流量）
firewall-cmd --panic-off     # 禁用
```

---

## 设置默认区域

```bash
firewall-cmd --set-default-zone=public
```

---

## 安装 firewalld

```bash
# Debian / Ubuntu
sudo apt install firewalld

# RHEL / CentOS
sudo yum install firewalld
```

---

## ⚠️ 注意事项

1. **`--permanent` 参数仅写入配置文件，不立即生效**，必须执行 `firewall-cmd --reload`
2. 未指定 `--zone=` 时，操作默认作用于默认区域（通常是 `public`）
3. **不要同时运行 firewalld 和 iptables（或 nftables）**，易导致规则冲突
4. CentOS 7+ 默认用 firewalld（后端可为 nftables 或 iptables）
5. 旧系统（如 CentOS 6）使用 `iptables` 命令，非 `firewall-cmd`
6. 查看帮助：`firewall-cmd --help`
