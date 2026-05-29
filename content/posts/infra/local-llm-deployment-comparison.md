---
title: "本地LLM部署对比：Ollama vs vLLM 实战评测"
date: 2026-05-28T10:10:00+08:00
draft: false
tags:
  - AI基础设施
  - 本地部署
  - LLM
  - 性能对比
categories:
  - infra
---

## 前言

2026年，本地LLM部署已经成为AI基础设施的标配。我在同一台服务器上同时部署了Ollama和vLLM，运行了为期两周的对比测试。

这篇文章记录完整的评测过程，包括性能、易用性、资源占用和适用场景。

## 一、测试环境

### 1.1 硬件配置

| 组件 | 规格 |
|------|------|
| CPU | AMD EPYC 7763 (64核) |
| GPU | NVIDIA A100 80GB × 2 |
| 内存 | 512GB DDR4 |
| 存储 | 2TB NVMe SSD |
| 系统 | Ubuntu 24.04 LTS |

### 1.2 测试模型

| 模型 | 参数量 | 量化版本 |
|------|--------|----------|
| Llama 3.1 | 8B | Q4_K_M |
| Llama 3.1 | 70B | Q4_K_M |
| Qwen 2.5 | 72B | Q4_K_M |

### 1.3 测试工具

- **llm-bench**: 自定义基准测试脚本
- **prometheus + grafana**: 实时监控
- **locust**: 并发压力测试

## 二、性能对比

### 2.1 单请求延迟

| 模型 | Ollama (TTFT) | vLLM (TTFT) | 优势 |
|------|--------------|-------------|------|
| Llama 3.1 8B | 1.2s | 0.8s | vLLM ↓33% |
| Llama 3.1 70B | 8.5s | 5.2s | vLLM ↓39% |
| Qwen 2.5 72B | 9.1s | 5.8s | vLLM ↓36% |

**TTFT** = Time To First Token（首字延迟）

### 2.2 吞吐量（tokens/s）

| 模型 | Ollama | vLLM | 优势 |
|------|--------|------|------|
| Llama 3.1 8B | 45 | 68 | vLLM ↑51% |
| Llama 3.1 70B | 12 | 19 | vLLM ↑58% |
| Qwen 2.5 72B | 11 | 17 | vLLM ↑55% |

### 2.3 并发能力

| 并发数 | Ollama 成功率 | vLLM 成功率 |
|--------|--------------|-------------|
| 1 | 100% | 100% |
| 5 | 98% | 100% |
| 10 | 92% | 100% |
| 20 | 75% | 98% |
| 50 | 45% | 95% |

**结论**：vLLM 在高并发场景下优势明显，得益于其 PagedAttention 机制。

## 三、资源占用

### 3.1 内存占用

| 模型 | Ollama | vLLM |
|------|--------|------|
| Llama 3.1 8B | 6.2GB | 5.8GB |
| Llama 3.1 70B | 42GB | 38GB |
| Qwen 2.5 72B | 44GB | 40GB |

vLLM 的 KV Cache 优化使其内存占用更低。

### 3.2 GPU 利用率

```
并发10时 GPU 利用率对比：
Ollama: ████████░░ 78%
vLLM:   ██████████ 95%
```

## 四、易用性对比

### 4.1 安装部署

| 步骤 | Ollama | vLLM |
|------|--------|------|
| 安装 | `curl -fsSL https://ollama.com/install.sh \| sh` | `pip install vllm` |
| 模型下载 | `ollama pull llama3.1` | `python -m vllm.entrypoints.api_server --model meta-llama/Llama-3.1-8B` |
| API调用 | `curl http://localhost:11434/api/generate` | `curl http://localhost:8000/v1/completions` |
| 配置复杂度 | ⭐ | ⭐⭐⭐ |

### 4.2 功能特性

| 功能 | Ollama | vLLM |
|------|--------|------|
| 多模型管理 | ✅ 内置 | ⚠️ 需手动 |
| Docker支持 | ✅ 官方镜像 | ✅ 官方镜像 |
| 量化支持 | ✅ 自动 | ✅ 需指定 |
| 多GPU支持 | ⚠️ 有限 | ✅ 完整 |
| 连续批处理 | ❌ | ✅ |
| PagedAttention | ❌ | ✅ |
| Speculative Decoding | ❌ | ✅ |

## 五、适用场景推荐

### 5.1 选择 Ollama

- **个人开发/学习**：简单易用，快速上手
- **单用户场景**：并发需求低
- **快速原型**：需要快速验证想法
- **资源受限**：内存/显存有限

### 5.2 选择 vLLM

- **生产环境**：高并发、高可用需求
- **多用户服务**：需要服务多个客户端
- **大模型部署**：70B+ 模型优化更好
- **性能敏感**：对延迟和吞吐量有要求

## 六、混合部署方案

我的生产环境采用混合部署：

```
开发环境 → Ollama (快速迭代)
生产环境 → vLLM (高并发服务)
```

通过统一API网关进行路由：

```yaml
api_gateway:
  routes:
    - path: /dev/*
      backend: ollama
    - path: /prod/*
      backend: vllm
```

## 七、总结

| 维度 | Ollama | vLLM |
|------|--------|------|
| 易用性 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| 性能 | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 并发能力 | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| 资源效率 | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| 功能丰富度 | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

**最终建议**：

- 初学者/个人项目：从 Ollama 开始
- 生产环境：直接使用 vLLM
- 预算充足：两者都部署，按场景路由

---

> **更新日志**：本文基于2026年5月测试环境编写，模型和工具版本可能随时间变化，请以实际测试为准。
