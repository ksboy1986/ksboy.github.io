---
title: "Docker Compose 多环境管理：从开发到生产的优雅方案"
date: 2026-05-28T10:20:00+08:00
draft: false
tags:
  - 运维实践
  - Docker
  - 多环境管理
  - CI/CD
categories:
  - ops
---

## 前言

2025年，我经历过三次因为环境不一致导致的线上故障。每次排查都花费数小时，最终发现是开发环境和生产环境的配置差异造成的。

从那时起，我开始系统性地重构多环境管理方案。这篇文章记录完整的实践过程，包括目录结构、配置管理和部署流程。

## 一、问题根源

### 1.1 常见痛点

| 问题 | 现象 | 影响 |
|------|------|------|
| 配置硬编码 | 环境变量写死在 docker-compose.yml | 切换环境需修改文件 |
| 镜像版本混乱 | 开发用 latest，生产用具体版本 | 行为不一致 |
| 依赖管理缺失 | 数据库迁移脚本未版本化 | 数据不一致 |
| 密钥管理不当 | 敏感信息明文存储 | 安全风险 |

### 1.2 根本原因

**环境隔离不彻底**：开发、测试、生产共用同一份配置模板，仅靠注释区分。

## 二、目录结构设计

### 2.1 推荐结构

```
project/
├── docker-compose.yml          # 基础配置（公共部分）
├── docker-compose.override.yml # 本地开发覆盖
├── environments/
│   ├── dev/
│   │   ├── docker-compose.dev.yml
│   │   └── .env.dev
│   ├── staging/
│   │   ├── docker-compose.staging.yml
│   │   └── .env.staging
│   └── prod/
│       ├── docker-compose.prod.yml
│       └── .env.prod
├── scripts/
│   ├── deploy.sh
│   └── rollback.sh
└── .gitignore
```

### 2.2 基础配置（docker-compose.yml）

```yaml
version: "3.8"

services:
  app:
    image: ${APP_IMAGE:-myapp:latest}
    restart: unless-stopped
    environment:
      - NODE_ENV=${NODE_ENV:-development}
      - LOG_LEVEL=${LOG_LEVEL:-info}
    depends_on:
      - db
      - redis

  db:
    image: postgres:${POSTGRES_VERSION:-16}
    volumes:
      - db_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=${POSTGRES_DB:-app}
      - POSTGRES_USER=${POSTGRES_USER:-app}
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password

  redis:
    image: redis:${REDIS_VERSION:-7-alpine}
    command: redis-server --maxmemory 256mb

volumes:
  db_data:
```

### 2.3 生产环境覆盖（environments/prod/docker-compose.prod.yml）

```yaml
version: "3.8"

services:
  app:
    image: myapp:${APP_VERSION:-1.0.0}
    deploy:
      resources:
        limits:
          cpus: "2"
          memory: 2G
        reservations:
          memory: 512M
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    secrets:
      - db_password
    networks:
      - frontend
      - backend

  db:
    deploy:
      resources:
        limits:
          memory: 4G
    volumes:
      - db_data:/var/lib/postgresql/data
      - ./backups:/backups

secrets:
  db_password:
    external: true

networks:
  frontend:
    driver: bridge
  backend:
    internal: true
```

## 三、环境变量管理

### 3.1 .env 文件规范

```bash
# .env.prod
# 应用配置
APP_VERSION=1.0.0
NODE_ENV=production
LOG_LEVEL=warn

# 数据库
POSTGRES_VERSION=16
POSTGRES_DB=app_prod
POSTGRES_USER=app

# Redis
REDIS_VERSION=7-alpine

# 镜像仓库
REGISTRY_URL=registry.example.com
```

### 3.2 密钥管理

**不要将密钥存入 .env 文件**！

```bash
# 使用 Docker secrets
echo "your-secure-password" | docker secret create db_password -

# 或在 Kubernetes 中使用 Secret
kubectl create secret generic db-credentials --from-literal=password=your-secure-password
```

## 四、部署脚本

### 4.1 部署脚本（scripts/deploy.sh）

```bash
#!/bin/bash
set -euo pipefail

ENV=${1:-dev}
PROJECT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"

echo "🚀 部署到环境: ${ENV}"

# 1. 加载环境变量
set -a
source "${PROJECT_DIR}/environments/${ENV}/.env.${ENV}"
set +a

# 2. 拉取最新镜像
docker compose -f docker-compose.yml \
               -f environments/${ENV}/docker-compose.${ENV}.yml \
               pull

# 3. 执行数据库迁移
docker compose -f docker-compose.yml \
               -f environments/${ENV}/docker-compose.${ENV}.yml \
               run --rm app npm run migrate

# 4. 启动服务
docker compose -f docker-compose.yml \
               -f environments/${ENV}/docker-compose.${ENV}.yml \
               up -d --remove-orphans

# 5. 健康检查
sleep 10
docker compose -f docker-compose.yml \
               -f environments/${ENV}/docker-compose.${ENV}.yml \
               ps

echo "✅ 部署完成"
```

### 4.2 回滚脚本（scripts/rollback.sh）

```bash
#!/bin/bash
set -euo pipefail

ENV=${1:-dev}
PROJECT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"

echo "⏪ 回滚环境: ${ENV}"

# 获取上一个版本
PREV_VERSION=$(docker images --format "{{.Tag}}" myapp | head -2 | tail -1)

# 更新环境变量
sed -i "s/APP_VERSION=.*/APP_VERSION=${PREV_VERSION}/" \
    "${PROJECT_DIR}/environments/${ENV}/.env.${ENV}"

# 重新部署
"${PROJECT_DIR}/scripts/deploy.sh" "${ENV}"

echo "✅ 回滚完成至版本: ${PREV_VERSION}"
```

## 五、CI/CD 集成

### 5.1 GitHub Actions 示例

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Staging
        run: ./scripts/deploy.sh staging
        env:
          DOCKER_REGISTRY_TOKEN: ${{ secrets.DOCKER_TOKEN }}

  deploy-prod:
    needs: deploy-staging
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Production
        run: ./scripts/deploy.sh prod
        env:
          DOCKER_REGISTRY_TOKEN: ${{ secrets.DOCKER_TOKEN }}
```

## 六、最佳实践总结

| 实践 | 说明 |
|------|------|
| ✅ 基础配置与覆盖分离 | docker-compose.yml 放公共配置，环境文件放差异 |
| ✅ 使用 .env 文件 | 不要硬编码环境变量 |
| ✅ 密钥使用 secrets | 不要将密钥存入版本控制 |
| ✅ 固定镜像版本 | 避免 latest 标签导致的不一致 |
| ✅ 健康检查 | 确保服务真正可用后再认为部署成功 |
| ✅ 回滚方案 | 每次部署前确认可以快速回滚 |
| ❌ 不要手动修改线上配置 | 所有变更通过代码审查 |
| ❌ 不要共享 .env 文件 | 每个环境独立文件 |

## 七、总结

多环境管理的核心原则：

1. **配置即代码**：所有环境配置版本化
2. **最小差异**：基础配置最大化，环境差异最小化
3. **自动化部署**：减少人为操作，提高一致性
4. **可回滚**：每次部署都有明确的回滚路径

如果你也在为环境不一致头疼，我的建议是：**尽早建立规范**，不要等到问题频发时才重构。

---

> **更新日志**：本文基于2026年5月实践编写，具体命令和配置可能因项目而异，请以实际需求为准。
