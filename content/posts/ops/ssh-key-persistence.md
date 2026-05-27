---
title: "SSH密钥持久化：为什么容器内生成的密钥在重启后丢失"
date: 2026-05-27
draft: false
tags:
  - SSH
  - 密钥管理
  - Docker
  - 运维实践
categories:
  - ops
---

## 前言

2026年5月，我遇到一个反复出现的问题：容器内生成的SSH密钥在容器重启后丢失，导致无法通过SSH连接到宿主机。

这个问题看似简单，但背后涉及Docker容器的文件系统隔离机制。这篇文章记录完整的排查过程和最终解决方案。

## 一、问题现象

```
现象：SSH密钥在容器内生成，容器重启后密钥消失，无法连接宿主机
时间：2026-05-18
环境：fnOS虚拟化平台 + Ubuntu 24.04 VM + Docker
```

**初始错误**：
```
Warning: Permanently added '192.168.0.200' (ED25519) to the list of known hosts.
Permission denied (publickey).
```

## 二、根因分析

### 2.1 Docker容器的文件系统隔离

Docker容器使用**联合文件系统（UnionFS）**，容器内的文件系统是独立的。当容器重启时：

1. **容器内生成的文件** → 存储在容器的可写层
2. **容器重启** → 可写层被销毁，所有未持久化的文件丢失
3. **SSH密钥丢失** → 无法通过密钥认证连接宿主机

### 2.2 为什么宿主机SSH拒绝使用容器内密钥

即使密钥被挂载到容器，宿主机SSH服务也会拒绝使用：

```
# /var/log/auth.log
sshd[12345]: Authentication refused: bad ownership or modes for key file
```

**原因**：SSH要求私钥文件所有者必须是 `root:root`，且权限为 `600`。容器内生成的密钥，挂载后文件所有者可能不匹配。

## 三、解决方案

### 3.1 核心原则

**密钥必须在宿主机生成，不能容器内生成。**

### 3.2 完整步骤

#### 步骤1：在宿主机生成SSH密钥

```bash
# 宿主机执行（192.168.0.200）
ssh-keygen -t ed25519 -C "hermes-agent" -f /home/ksboy/.ssh/hermes_key
# 设置权限
chmod 600 /home/ksboy/.ssh/hermes_key
chmod 644 /home/ksboy/.ssh/hermes_key.pub
```

#### 步骤2：将公钥添加到宿主机授权文件

```bash
# 宿主机执行
cat /home/ksboy/.ssh/hermes_key.pub >> /home/ksboy/.ssh/authorized_keys
chmod 600 /home/ksboy/.ssh/authorized_keys
```

#### 步骤3：在docker-compose.yml中挂载密钥

```yaml
services:
  hermes-agent:
    image: hermes-agent:latest
    volumes:
      # 密钥挂载（只读模式）
      - /home/ksboy/.ssh/hermes_key:/root/.ssh/id_ed25519:ro
      - /home/ksboy/.ssh/hermes_key.pub:/root/.ssh/id_ed25519.pub:ro
      # SSH配置
      - /home/ksboy/.ssh/config:/root/.ssh/config:ro
    environment:
      - SSH_HOST=192.168.0.200
      - SSH_USER=ksboy
```

#### 步骤4：宿主机SSH配置调整

```bash
# /etc/ssh/sshd_config
# 允许Docker网段访问
ListenAddress 0.0.0.0

# 重启SSH服务
systemctl restart sshd
```

### 3.3 验证

```bash
# 容器内测试
ssh -i /root/.ssh/id_ed25519 ksboy@192.168.0.200 "echo '连接成功'"

# 重启容器后再次测试
docker-compose restart hermes-agent
ssh -i /root/.ssh/id_ed25519 ksboy@192.168.0.200 "echo '重启后连接成功'"
```

## 四、关键要点

| 要点 | 说明 |
|------|------|
| 密钥生成位置 | **必须在宿主机**，容器内生成的密钥重启后丢失 |
| 文件所有者 | 宿主机密钥必须为 `root:root`（容器以root运行） |
| 挂载模式 | 使用 `:ro` 只读模式，防止容器内意外修改 |
| SSH监听地址 | 宿主机需监听 `0.0.0.0`，允许Docker网段访问 |
| 网络隔离 | 容器在Docker网段，宿主机在LAN网段，需正确配置路由 |

## 五、常见错误

### 错误1：容器内生成密钥

```bash
# ❌ 错误做法
docker exec -it hermes-agent ssh-keygen -t ed25519 -f /root/.ssh/id_ed25519
# 容器重启后密钥丢失
```

### 错误2：密钥权限不正确

```bash
# ❌ 错误做法
chmod 644 /home/ksboy/.ssh/hermes_key  # SSH拒绝使用
# ✅ 正确做法
chmod 600 /home/ksboy/.ssh/hermes_key
```

### 错误3：宿主机SSH只监听localhost

```bash
# ❌ 错误做法
ListenAddress 127.0.0.1  # Docker容器无法连接
# ✅ 正确做法
ListenAddress 0.0.0.0
```

## 六、总结

SSH密钥持久化的核心是理解Docker的文件系统隔离机制：

1. **容器内文件不是持久的** → 密钥必须在宿主机生成
2. **权限必须匹配** → 宿主机密钥所有者需与容器运行用户一致
3. **网络必须可达** → 宿主机SSH需监听所有地址

这个解决方案已经稳定运行超过2周，容器重启后SSH连接正常。

---

> **相关文档**：[SSH密钥持久化技能](~/.hermes/skills/git-credential-persistence/SKILL.md)
