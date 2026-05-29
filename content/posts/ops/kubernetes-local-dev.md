---
title: "Kubernetes 本地开发环境搭建：从0到1的完整指南"
date: 2026-05-28T10:30:00+08:00
draft: false
tags:
  - 运维实践
  - Kubernetes
  - 本地开发
  - DevOps
categories:
  - ops
---

## 前言

2025年之前，我的本地开发环境一直是 Docker Compose。直到一次生产环境的配置差异导致严重故障，我才意识到本地环境需要更接近生产。

这篇文章记录完整的 Kubernetes 本地开发环境搭建过程，包括工具选择、配置优化和开发工作流。

## 一、为什么需要本地 K8s

### 1.1 痛点分析

| 场景 | Docker Compose | Kubernetes |
|------|---------------|------------|
| ConfigMap 测试 | ❌ 不支持 | ✅ 原生支持 |
| Service 发现 | ⚠️ 手动配置 | ✅ 自动发现 |
| Ingress 路由 | ❌ 不支持 | ✅ 原生支持 |
| HPA 自动扩缩容 | ❌ 不支持 | ✅ 原生支持 |
| 生产一致性 | ⚠️ 较低 | ✅ 高 |

### 1.2 核心价值

**"本地即生产"**：在本地就能验证生产环境的配置和行为，减少部署时的意外。

## 二、工具选择

### 2.1 主流方案对比

| 工具 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| Minikube | 功能完整、插件丰富 | 启动慢、资源占用高 | 学习/测试 |
| Kind | 快速启动、Docker后端 | 多集群管理弱 | 开发/CI |
| K3s | 轻量、生产级 | 配置稍复杂 | 边缘/开发 |
| Docker Desktop K8s | 一键启用、集成好 | 资源占用高、Mac/Win独占 | 快速上手 |
| Rancher Desktop | 跨平台、可选容器运行时 | 较新、社区较小 | 跨平台开发 |

### 2.2 我的选择：Kind

经过对比测试，我选择 **Kind (Kubernetes in Docker)** 作为本地开发环境：

- ✅ 启动速度快（~30秒）
- ✅ 资源占用低（~2GB内存）
- ✅ 多集群支持（开发/测试环境隔离）
- ✅ 与 CI/CD 一致（GitHub Actions 也用 Kind）

## 三、环境搭建

### 3.1 安装工具

```bash
# 安装 Docker
curl -fsSL https://get.docker.com | sh

# 安装 Kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# 安装 kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# 安装 Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### 3.2 创建集群

```bash
# 创建开发集群
kind create cluster --name dev --config kind-config.yaml

# 创建测试集群（隔离环境）
kind create cluster --name test --config kind-config.yaml
```

### 3.3 集群配置（kind-config.yaml）

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
  - role: worker
  - role: worker
```

## 四、核心组件部署

### 4.1 Ingress Controller

```bash
# 部署 NGINX Ingress
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# 验证
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

### 4.2 本地 DNS

```bash
# 安装 CoreDNS 优化配置
kubectl apply -f https://raw.githubusercontent.com/coredns/coredns/master/coredns.yaml
```

### 4.3 存储类

```yaml
# local-path-storage.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
```

```bash
kubectl apply -f local-path-storage.yaml
```

## 五、开发工作流

### 5.1 镜像构建与加载

```bash
# 使用 Kind 内置的 Docker  Registry
kind build node-image --image myapp:dev ./

# 或直接加载到集群
kind load docker-image myapp:dev --name dev
```

### 5.2 热重载开发

使用 **Telepresence** 实现本地代码热重载：

```bash
# 安装 Telepresence
brew install telepresence  # macOS
# 或
curl -fL https://app.gettelepresence.io/download/linux/binary > telepresence && chmod +x telepresence && sudo mv telepresence /usr/local/bin/

# 拦截服务流量
telepresence intercept myapp --port 3000:3000
```

### 5.3 端口转发

```bash
# 临时端口转发
kubectl port-forward svc/myapp 3000:3000 -n dev

# 或使用 kubectl-aliases 简化
alias kpf='kubectl port-forward'
kpf svc/myapp 3000:3000
```

## 六、配置管理

### 6.1 ConfigMap 示例

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: dev
data:
  NODE_ENV: "development"
  LOG_LEVEL: "debug"
  API_ENDPOINT: "http://api.dev.local"
```

```bash
kubectl apply -f configmap.yaml
```

### 6.2 Secret 管理

```bash
# 创建 Secret
kubectl create secret generic db-credentials \
  --from-literal=username=app \
  --from-literal=password=secret \
  -n dev

# 或使用 Helm Secrets 插件
helm secrets install my-release ./charts/myapp \
  --set db.password=$(cat .secrets/db-password)
```

## 七、调试技巧

### 7.1 快速查看日志

```bash
# 查看 Pod 日志
kubectl logs -f deployment/myapp -n dev

# 查看上一个实例的日志（重启后）
kubectl logs -f deployment/myapp -n dev --previous

# 查看特定容器
kubectl logs -f deployment/myapp -c sidecar -n dev
```

### 7.2 进入容器调试

```bash
# 进入容器
kubectl exec -it deployment/myapp -n dev -- /bin/sh

# 或使用 debug 模式启动临时容器
kubectl debug -it deployment/myapp -n dev --image=busybox --target=myapp
```

### 7.3 资源监控

```bash
# 查看资源使用
kubectl top pods -n dev
kubectl top nodes

# 查看事件
kubectl get events -n dev --sort-by='.lastTimestamp'
```

## 八、CI/CD 集成

### 8.1 GitHub Actions 示例

```yaml
# .github/workflows/test.yml
name: Test

on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Kind
        uses: helm/kind-action@v1
        with:
          config: kind-config.yaml
      
      - name: Deploy
        run: |
          kubectl apply -f k8s/
          kubectl wait --for=condition=ready pod -l app=myapp --timeout=120s
      
      - name: Run Tests
        run: npm test
```

## 九、总结

本地 Kubernetes 开发环境的核心价值：

1. **一致性**：本地行为接近生产，减少部署意外
2. **快速迭代**：启动快、资源占用低
3. **完整功能**：支持 ConfigMap、Ingress、HPA 等 K8s 原生特性
4. **CI/CD 一致**：本地和 CI 使用相同工具链

**推荐配置**：

| 场景 | 推荐工具 |
|------|----------|
| 快速上手 | Docker Desktop K8s |
| 日常开发 | Kind |
| 多集群隔离 | Kind + 多个集群 |
| 生产预演 | K3s |

---

> **更新日志**：本文基于2026年5月实践编写，工具版本可能随时间变化，请以官方文档为准。
