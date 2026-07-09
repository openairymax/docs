# Airymax 部署指南

> **文档编号**: P4.3.7  
> **版本**: 0.1.1  
> **适用范围**: Airymax v0.1.1  
> **最后更新**: 2026-06-18

---

## 目录

1. [部署架构概览](#1-部署架构概览)
2. [硬件需求](#2-硬件需求)
3. [Docker 部署](#3-docker-部署)
4. [Kubernetes 部署](#4-kubernetes-部署)
5. [配置管理](#5-配置管理)
6. [Daemon 启动顺序与健康检查](#6-daemon-启动顺序与健康检查)
7. [数据库设置](#7-数据库设置)
8. [监控设置](#8-监控设置)
9. [日志与追踪设置](#9-日志与追踪设置)
10. [备份与灾难恢复](#10-备份与灾难恢复)
11. [安全加固](#11-安全加固)
12. [生产环境检查清单](#12-生产环境检查清单)

---

## 1. 部署架构概览

Airymax 是一个多 daemon 协作的智能 Agent 运行时平台，采用分层 DAG 架构设计。系统支持三种部署模式：

### 1.1 单节点部署（Single-Node）

适用于开发、测试和小规模生产环境。所有 daemon 和基础设施服务（Redis、PostgreSQL）运行在同一台主机上，通过 Docker Compose 或 systemd 编排。

```
┌─────────────────────────────────────────────────────┐
│                    单节点主机                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │ Gateway  │  │  LLM_D   │  │ Tool_D   │  ...      │
│  │  :8080   │  │  :9201   │  │  :9202   │           │
│  └──────────┘  └──────────┘  └──────────┘           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │  Redis   │  │PostgreSQL│  │  Jaeger  │           │
│  │  :6379   │  │  :5432   │  │  :16686  │           │
│  └──────────┘  └──────────┘  └──────────┘           │
└─────────────────────────────────────────────────────┘
```

### 1.2 多节点部署（Multi-Node）

适用于中等规模生产环境。将 daemon 按层级分散到多台主机，基础设施服务独立部署。

- **节点 A（业务层）**: gateway_d, llm_d, tool_d, market_d
- **节点 B（核心层）**: corekern, coreloopthree, taskflow, memory, sched_d, channel_d
- **节点 C（基础设施层）**: monit_d, observe_d, info_d, notify_d
- **节点 D（数据层）**: Redis, PostgreSQL
- **节点 E（可观测性）**: Prometheus, Grafana, Jaeger, OpenTelemetry Collector

### 1.3 Kubernetes 部署

适用于大规模生产环境，利用 K8s 的自动扩缩容、服务发现和自愈能力。使用 Helm Chart 进行部署管理。

```
┌──────────────────────────────────────────────────────────┐
│                    Kubernetes 集群                         │
│  ┌──────────────────┐  ┌──────────────────┐               │
│  │   Ingress (Nginx) │  │  cert-manager    │               │
│  │   :80 / :443      │  │  (TLS 自动续期)  │               │
│  └────────┬─────────┘  └──────────────────┘               │
│           │                                                │
│  ┌────────▼─────────────────────────────────────────┐     │
│  │              Gateway Service (LoadBalancer)        │     │
│  │              HTTP :8080 / WS :8081 / Metrics :9090 │     │
│  └────────┬─────────────────────────────────────────┘     │
│           │                                                │
│  ┌────────▼─────────────────────────────────────────┐     │
│  │  Daemon Services (ClusterIP)                       │     │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐   │     │
│  │  │llm-d │ │tool-d│ │sched │ │market│ │memory│   │     │
│  │  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘   │     │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐            │     │
│  │  │monit │ │obsv  │ │plugin│ │hook  │  ...       │     │
│  │  └──────┘ └──────┘ └──────┘ └──────┘            │     │
│  └──────────────────────────────────────────────────┘     │
│  ┌──────────────────┐  ┌──────────────────────────┐      │
│  │  Redis (主从)     │  │  PostgreSQL (主+只读副本) │      │
│  └──────────────────┘  └──────────────────────────┘      │
│  ┌──────────────────┐  ┌──────────────────────────┐      │
│  │  Prometheus      │  │  Grafana                  │      │
│  │  + AlertManager  │  │  (Dashboard 可视化)       │      │
│  └──────────────────┘  └──────────────────────────┘      │
└──────────────────────────────────────────────────────────┘
```

---

## 2. 硬件需求

### 2.1 最低配置（开发/测试环境）

| 资源 | 要求 | 说明 |
|------|------|------|
| **CPU** | 4 核 (x86_64) | 至少 2.0 GHz |
| **内存** | 8 GB | 包含所有 daemon + Redis + PostgreSQL |
| **磁盘** | 50 GB SSD | 用于日志、数据库持久化和容器镜像 |
| **网络** | 1 Gbps | 内网通信 |

### 2.2 推荐配置（生产环境单节点）

| 资源 | 要求 | 说明 |
|------|------|------|
| **CPU** | 16 核 (x86_64) | 推荐 2.5 GHz 以上 |
| **内存** | 32 GB | daemon 8 GB + PostgreSQL 8 GB + Redis 3 GB + 余量 |
| **磁盘** | 200 GB NVMe SSD | 数据库 50 GB + 日志 50 GB + 镜像 50 GB + 余量 |
| **网络** | 10 Gbps | 低延迟内网 |

### 2.3 生产环境各组件资源限制

参考 `deploy/docker/docker-compose.prod.yml` 中的生产配置：

| 组件 | CPU Limit | Memory Limit | 副本数 |
|------|-----------|-------------|--------|
| agentos-atoms (内核) | 4.0 | 8 GB | 2 |
| agentos-gateway (网关) | 2.0 | 4 GB | 3 |
| agentos-daemon (服务层) | 4.0 | 8 GB | 2 |
| agentos-openlab (API) | 2.0 | 4 GB | 3 |
| Redis | 1.0 | 3 GB | 1 |
| PostgreSQL | 2.0 | 8 GB | 1 |
| Jaeger | 1.0 | 2 GB | 1 |
| Prometheus | 1.0 | 4 GB | 1 |
| Grafana | 1.0 | 2 GB | 1 |
| Nginx | 0.5 | 512 MB | 1 |

### 2.4 Kubernetes 节点建议

| 节点类型 | 数量 | 规格 | 用途 |
|---------|------|------|------|
| Control Plane | 3 | 4C / 8G | K8s 控制面 |
| Worker (业务) | 3+ | 16C / 32G | daemon Pod |
| Worker (数据) | 2 | 8C / 32G | Redis + PostgreSQL |
| Worker (监控) | 1 | 4C / 8G | Prometheus + Grafana + Jaeger |

---

## 3. Docker 部署

Airymax 提供了两套 Docker 部署方案：
- **`deploy/docker/`**：全系统多服务编排（内核 + 服务层 + 基础设施）

以下以 `deploy/docker/` 全系统方案为例。

### 3.1 前置条件

- Docker Engine ≥ 20.10
- Docker Compose ≥ v2.0
- 至少 8 GB 可用内存

### 3.2 快速启动

```bash
# 进入部署目录
cd deploy/docker

# 一键部署（环境检查 → 镜像构建 → 服务启动）
./quickstart.sh
```

### 3.3 手动部署步骤

**Step 1: 配置环境变量**

```bash
cd deploy/docker
cp configs/env.example .env
# 编辑 .env，填入实际 API 密钥和数据库密码
vim .env
```

关键环境变量（`configs/env.example`）：

```ini
# API 密钥
OPENAI_API_KEY=sk-your-openai-api-key-here
DEEPSEEK_API_KEY=sk-your-deepseek-api-key-here
ANTHROPIC_API_KEY=sk-ant-your-anthropic-api-key-here

# 数据库密码
POSTGRES_PASSWORD=your_secure_password

# 日志级别
AGENTOS_LOG_LEVEL=INFO
AGENTOS_LOG_FORMAT=text

# 追踪
AGENTOS_ENABLE_TRACING=true
AGENTOS_TELEMETRY_ENDPOINT=http://otel-collector:4317
```

**Step 2: 构建镜像**

```bash
# 生产版本（精简镜像）
make build-all

# 或开发版本（包含调试工具）
make build-all-dev
```

**Step 3: 启动服务**

```bash
# 开发环境
make start

# 预览环境
docker-compose -f docker-compose.yml -f docker-compose.preview.yml up -d

# 预发布环境
docker-compose -f docker-compose.yml -f docker-compose.staging.yml up -d

# 生产环境
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

**Step 4: 验证部署**

```bash
# 查看服务状态
make ps

# 检查健康状态
make health

# 查看日志
make logs
```

### 3.4 多环境编排对比

| 环境 | 编排文件 | 特点 |
|------|---------|------|
| 开发 (dev) | `docker-compose.yml` | 端口直接暴露，Debug 日志，无资源限制 |
| 预览 (preview) | `docker-compose.preview.yml` | 基本资源限制和健康检查 |
| 预发布 (staging) | `docker-compose.staging.yml` | 配置与生产一致 |
| 生产 (prod) | `docker-compose.prod.yml` | 高可用、安全加固、监控完备、资源限制 |

### 3.5 生产环境 Compose 示例

以下是 `docker-compose.prod.yml` 中的关键生产配置：

```yaml
services:
  agentos-atoms:
    image: ghcr.io/spharx/agentos-atoms:0.1.0
    restart: always
    deploy:
      resources:
        limits:
          cpus: '4.0'
          memory: 8G
        reservations:
          cpus: '2.0'
          memory: 4G
      replicas: 2
    environment:
      - AGENTOS_ENV=production
      - AGENTOS_LOG_LEVEL=WARN
      - AGENTOS_LOG_FORMAT=json
      - AGENTOS_ENABLE_TRACING=true
      - AGENTOS_ENABLE_METRICS=true
      - AGENTOS_SECURITY_LEVEL=high
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8091/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "10"
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL

  agentos-gateway:
    image: ghcr.io/spharx/agentos-gateway:0.1.0
    restart: always
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '2.0'
          memory: 4G
    ports:
      - "127.0.0.1:8080:8080"
      - "127.0.0.1:8443:8443"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 45s

  postgres-prod:
    image: postgres:15-alpine
    restart: always
    command: >
      postgres
      -c max_connections=200
      -c shared_buffers=512MB
      -c effective_cache_size=1GB
      -c maintenance_work_mem=128MB
      -c wal_buffers=16MB
      -c work_mem=4MB
      -c min_wal_size=1GB
      -c max_wal_size=4GB
    environment:
      - POSTGRES_DB=agentos_prod
      - POSTGRES_USER=agentos
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD:?POSTGRES_PASSWORD is required}
      - POSTGRES_INITDB_ARGS=--encoding=UTF-8 --locale=C
    volumes:
      - postgres-prod-data:/var/lib/postgresql/data
      - postgres-backups:/var/lib/postgresql/backups
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U agentos -d agentos_prod"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 60s

  redis-prod:
    image: redis:7-alpine
    restart: always
    command: >
      redis-server
      --appendonly yes
      --maxmemory 2gb
      --maxmemory-policy allkeys-lru
    volumes:
      - redis-prod-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  nginx-prod:
    image: nginx:1.27-alpine
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./ecosystem/manager/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ecosystem/manager/nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - agentos-gateway
      - agentos-openlab
    security_opt:
      - no-new-privileges:true
    read_only: true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE

volumes:
  prod-atoms-logs:    { driver: local }
  prod-atoms-data:    { driver: local }
  prod-gateway-logs:  { driver: local }
  prod-daemon-logs:   { driver: local }
  prod-daemon-data:   { driver: local }
  prod-openlab-logs:  { driver: local }
  redis-prod-data:    { driver: local }
  postgres-prod-data: { driver: local }
  postgres-backups:   { driver: local }
  jaeger-data:        { driver: local }
  prometheus-data:    { driver: local }
  grafana-data:       { driver: local }
  nginx-cache:        { driver: local }
  nginx-logs:         { driver: local }

networks:
  agentos-production-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.30.0.0/16
  agentos-internal-network:
    driver: bridge
    internal: false
    ipam:
      config:
        - subnet: 172.31.0.0/16
```

### 3.6 Makefile 快捷命令

```bash
make build-kernel       # 构建内核镜像
make build-service      # 构建服务镜像
make build-all          # 构建所有镜像
make start              # 启动所有服务
make stop               # 停止所有服务
make restart            # 重启所有服务
make down               # 停止并删除容器
make clean              # 清理所有容器、镜像和数据卷
make logs               # 查看所有日志
make logs-kernel        # 查看内核日志
make logs-service       # 查看服务日志
make ps                 # 查看服务状态
make health             # 检查服务健康状态
make shell-kernel       # 进入内核容器
make shell-service      # 进入服务容器
make backup             # 备份数据卷
make stats              # 查看资源使用情况
```

---

## 4. Kubernetes 部署

### 4.1 前置条件

- Kubernetes 集群 ≥ v1.22
- Helm ≥ v3.8
- kubectl 已配置集群访问
- 集群中有可用的 StorageClass（用于 PVC 持久化）

### 4.2 Helm Chart 结构

```
deploy/kubernetes/helm/
├── Chart.yaml              # Chart 元信息 (v0.1.1)
├── values.yaml             # 默认配置值
├── values-prod.yaml        # 生产环境覆盖配置
└── templates/
    ├── _helpers.tpl        # 模板辅助函数
    ├── configmap.yaml      # ConfigMap 配置
    ├── deployments.yaml    # 10 个 Daemon 用户态服务 + 核心模块 Deployment
    ├── ingress.yaml        # Ingress 入口
    └── services.yaml       # 10 个 Daemon 用户态服务 + 核心模块 Service
```

### 4.3 安装 Chart

```bash
# 使用默认配置安装
helm install agentrt ./deploy/kubernetes/helm \
  --namespace agentrt \
  --create-namespace

# 使用生产配置安装（叠加）
helm install agentrt ./deploy/kubernetes/helm \
  -f ./deploy/kubernetes/helm/values.yaml \
  -f ./deploy/kubernetes/helm/values-prod.yaml \
  --namespace agentrt \
  --create-namespace

# 指定自定义参数
helm install agentrt ./deploy/kubernetes/helm \
  --set global.image.tag=0.1.1 \
  --set gateway_d.replicaCount=3 \
  --set ingress.enabled=true \
  --set ingress.hosts[0].host=agentrt.example.com \
  --namespace agentrt
```

### 4.4 部署的 Daemon 列表

Helm Chart 部署 10 个 Daemon 用户态服务 + 核心模块，分为四个层级：

| 层级 | 类别 | Daemon | 默认端口 | 生产副本数 |
|------|------|--------|---------|-----------|
| 内核原子 | 核心模块 | corekern | 9001 | 3 |
| 内核原子 | 核心模块 | coreloopthree | 9002 | 3 |
| 内核原子 | 核心模块 | taskflow | 9003 | 2 |
| 内核原子 | 核心模块 | memory | 9004 | 2 |
| 基础 daemon | 基础设施 | channel_d | 9101 | 2 |
| 基础 daemon | 基础设施 | monit_d | 9102 | 2 |
| 基础 daemon | 基础设施 | observe_d | 9103 | 2 |
| 业务 daemon | 业务 | llm_d | 9201 | 2 |
| 业务 daemon | 业务 | tool_d | 9202 | 2 |
| 业务 daemon | 业务 | market_d | 9203 | 2 |
| 业务 daemon | 业务 | sched_d | 9204 | 2 |
| 扩展 daemon | 扩展 | info_d | 9303 | 2 |
| 扩展 daemon | 扩展 | notify_d | 9304 | 2 |
| 网关层 | 网关 | gateway_d | 8080/8081/9090 | 3 |

### 4.5 生产环境 values-prod.yaml 关键配置

```yaml
global:
  env: production
  image:
    pullPolicy: Always
  replicaCount: 3

# Redis 主从模式
redis:
  enabled: true
  architecture: replication
  auth:
    enabled: true
    existingSecret: agentrt-redis-secret
  master:
    persistence:
      enabled: true
      size: 20Gi
  replica:
    replicaCount: 2
    persistence:
      enabled: true
      size: 20Gi

# PostgreSQL 主从模式
postgresql:
  enabled: true
  auth:
    existingSecret: agentrt-postgresql-secret
  primary:
    persistence:
      enabled: true
      size: 50Gi
  readReplicas:
    replicaCount: 1

# 网关（生产环境）
gateway_d:
  replicaCount: 3
  resources:
    requests:
      memory: "256Mi"
      cpu: "200m"
    limits:
      memory: "1Gi"
      cpu: "1000m"

# Ingress（生产环境）
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
  hosts:
    - host: agentrt.example.com
      paths:
        - path: /
          pathType: Prefix
          port: 9000
  tls:
    - secretName: agentrt-tls
      hosts:
        - agentrt.example.com

# Pod 反亲和性（避免单点）
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
              - key: app.kubernetes.io/name
                operator: In
                values:
                  - agentrt
          topologyKey: kubernetes.io/hostname
```

### 4.6 ConfigMap 配置

Helm Chart 自动生成统一的 ConfigMap，包含所有 daemon 的服务发现地址：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: agentrt-config
data:
  agentos.yaml: |
    agentos:
      version: "0.1.1"
      log_level: "INFO"
      deploy_env: "production"

      corekern:
        host: agentrt-corekern
        port: 9001

      coreloopthree:
        host: agentrt-coreloopthree
        port: 9002

      daemons:
        channel:
          host: agentrt-channel-d
          port: 9101
        monit:
          host: agentrt-monit-d
          port: 9102
        llm:
          host: agentrt-llm-d
          port: 9201
        tool:
          host: agentrt-tool-d
          port: 9202
        market:
          host: agentrt-market-d
          port: 9203
        sched:
          host: agentrt-sched-d
          port: 9204
        # ... 其他 daemon

      gateway:
        http_port: 8080
        ws_port: 8081
        metrics_port: 9090
        mcp_enabled: true
        a2a_enabled: true
        openai_enabled: true

      infra:
        redis:
          host: agentrt-redis-master
          port: 6379
        postgres:
          host: agentrt-postgresql
          port: 5432
          database: agentrt
          username: agentrt
```

### 4.7 升级与回滚

```bash
# 升级
helm upgrade agentrt ./deploy/kubernetes/helm \
  -f ./deploy/kubernetes/helm/values-prod.yaml

# 查看历史
helm history agentrt -n agentrt

# 回滚到上一个版本
helm rollback agentrt -n agentrt

# 回滚到指定版本
helm rollback agentrt 3 -n agentrt
```

### 4.8 卸载

```bash
helm uninstall agentrt -n agentrt
# 注意：PVC 不会自动删除，需要手动清理
kubectl delete pvc -n agentrt -l app.kubernetes.io/instance=agentrt
```

---

## 5. 配置管理

### 5.1 环境变量

Airymax 通过环境变量进行主要配置。完整的变量列表参考 `configs/env.example`：

#### 核心配置

| 变量 | 说明 | 默认值 | 生产建议 |
|------|------|--------|---------|
| `AGENTOS_ENV` | 运行环境 | `development` | `production` |
| `AGENTOS_LOG_LEVEL` | 日志级别 | `INFO` | `WARN` |
| `AGENTOS_LOG_FORMAT` | 日志格式 | `text` | `json` |
| `AGENTOS_ENABLE_TRACING` | 启用追踪 | `true` | `true` |
| `AGENTOS_ENABLE_METRICS` | 启用指标 | `true` | `true` |
| `AGENTOS_TELEMETRY_ENDPOINT` | OTLP 端点 | `http://otel-collector:4317` | 同左 |
| `AGENTOS_DEPLOYMENT_TYPE` | 部署类型 | - | `production` |
| `AGENTOS_SECURITY_LEVEL` | 安全级别 | - | `high` |
| `TZ` | 时区 | `UTC` | `Asia/Shanghai` |

#### API 密钥

| 变量 | 说明 |
|------|------|
| `OPENAI_API_KEY` | OpenAI API 密钥 |
| `DEEPSEEK_API_KEY` | DeepSeek API 密钥 |
| `ANTHROPIC_API_KEY` | Anthropic (Claude) API 密钥 |

#### 数据库

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `POSTGRES_PASSWORD` | PostgreSQL 密码 | `agentos_dev`（开发） |
| `POSTGRES_DB` | 数据库名 | `agentos` |
| `POSTGRES_USER` | 数据库用户 | `agentos` |
| `REDIS_HOST` | Redis 主机 | `redis` |
| `REDIS_PORT` | Redis 端口 | `6379` |

#### 服务端口

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `LLM_SERVICE_PORT` | 8080 | LLM 服务端口 |
| `TOOL_SERVICE_PORT` | 8081 | 工具服务端口 |
| `MARKET_SERVICE_PORT` | 8082 | 市场服务端口 |
| `SCHED_SERVICE_PORT` | 8083 | 调度服务端口 |
| `MONIT_SERVICE_PORT` | 8084 | 监控服务端口 |

#### 性能调优

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `AGENTOS_MAX_WORKERS` | 8 | 最大工作进程数 |
| `LLM_REQUEST_TIMEOUT` | 120 | LLM 请求超时（秒） |
| `AGENTOS_CACHE_SIZE_MB` | 512 | 缓存大小（MB） |
| `MEMORY_LRU_CACHE_SIZE` | 10000 | LRU 缓存条目数 |
| `MEMORY_BATCH_SIZE` | 100 | 记忆写入批处理大小 |

#### 安全

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `AGENTOS_JWT_SECRET` | - | JWT 密钥（生产必须修改） |
| `API_RATE_LIMIT` | 100 | 每分钟请求数限制 |
| `ENABLE_CORS` | true | 启用 CORS |
| `CORS_ALLOWED_ORIGINS` | `http://localhost:3000` | 允许的 CORS 源 |

### 5.2 配置文件

Airymax 使用 YAML 格式的配置文件管理 daemon 间服务发现和参数：

- **内核配置**: `ecosystem/manager/kernel/kernel.yaml`
- **服务配置**: `ecosystem/manager/services/`
- **OpenTelemetry 配置**: `ecosystem/manager/otel-collector-manager.yaml`
- **Prometheus 配置**: `ecosystem/manager/prometheus.yml`
- **Nginx 配置**: `ecosystem/manager/nginx/nginx.conf`

### 5.3 配置检查

部署前使用配置检查脚本验证环境：

```bash
cd deploy/docker
./check_config.sh
```

该脚本检查项包括：
- Docker 环境是否就绪
- 必要文件是否存在（Dockerfile, docker-compose.yml 等）
- `.env` 文件是否配置且密钥非默认值
- 端口占用情况（8080-8084）
- docker-compose.yml 语法正确性
- 资源限制是否配置
- 健康检查是否配置
- 数据卷持久化是否配置
- 脚本执行权限

### 5.4 Secrets 管理

密钥文件存放在 `deploy/docker/secrets/` 目录：

```bash
deploy/docker/secrets/
└── .gitkeep    # 保持目录结构，实际密钥文件不应提交到 Git
```

**Docker 部署**：通过 `.env` 文件注入敏感信息，`.env` 已在 `.gitignore` 中排除。

**Kubernetes 部署**：通过 K8s Secret 管理：

```bash
# 创建 PostgreSQL 密码 Secret
kubectl create secret generic agentrt-postgresql-secret \
  --from-literal=password='your_secure_password' \
  -n agentrt

# 创建 Redis 密码 Secret
kubectl create secret generic agentrt-redis-secret \
  --from-literal=redis-password='your_redis_password' \
  -n agentrt

# 创建 API 密钥 Secret
kubectl create secret generic agentrt-api-keys \
  --from-literal=openai-api-key='sk-...' \
  --from-literal=deepseek-api-key='sk-...' \
  -n agentrt
```

---

## 6. Daemon 启动顺序与健康检查

### 6.1 启动 DAG

Airymax 采用 5 层 DAG 启动顺序，每层内 daemon 可并行启动，跨层必须等待前层健康检查通过：

```
Layer 0（基础设施 - 无依赖，并行启动）:
  ├── monit_d     (监控用户态服务)
  ├── observe_d   (可观测性用户态服务)
  ├── info_d      (信息用户态服务)
  └── notify_d    (通知用户态服务)

Layer 1（核心服务 - 依赖 Layer 0）:
  ├── sched_d     (调度用户态服务 → 依赖 observe_d)
  └── channel_d   (通道用户态服务)

Layer 2（Agent 服务 - 依赖 Layer 1）:
  ├── llm_d       (LLM 用户态服务 → 依赖 sched_d)
  └── tool_d      (工具用户态服务)

Layer 3（业务服务 - 依赖 Layer 2）:
  └── market_d    (市场用户态服务)

Layer 4（网关 - 依赖 Layer 2 和 Layer 3）:
  └── gateway_d   (网关用户态服务 → 依赖 llm_d, tool_d, market_d)
```

### 6.2 systemd 启动顺序

在单节点部署中，systemd service 文件实现了相同的依赖关系：

```ini
# agentrt-monit.service — Layer 0
[Unit]
Description=Airymax Monitor Daemon
After=network.target

# agentrt-sched.service — Layer 1
[Unit]
Description=Airymax Scheduler Daemon
After=network.target agentrt-observe.service
Requires=agentrt-observe.service

# agentrt-llm.service — Layer 2
[Unit]
Description=Airymax LLM Daemon
After=network.target agentrt-sched.service
Requires=agentrt-sched.service

# agentrt-gateway.service — Layer 4
[Unit]
Description=Airymax Gateway Daemon
After=network.target agentrt-llm.service agentrt-tool.service agentrt-market.service
Requires=agentrt-llm.service agentrt-tool.service agentrt-market.service
```

完整 target 文件 `agentrt.target`：

```ini
[Unit]
Description=Airymax Full Service Stack
After=agentrt-monit.service agentrt-observe.service agentrt-info.service agentrt-notify.service
After=agentrt-sched.service agentrt-channel.service
After=agentrt-llm.service agentrt-tool.service agentrt-hook.service agentrt-plugin.service
After=agentrt-market.service
After=agentrt-gateway.service
Wants=agentrt-monit.service agentrt-observe.service agentrt-info.service agentrt-notify.service
Wants=agentrt-sched.service agentrt-channel.service
Wants=agentrt-llm.service agentrt-tool.service agentrt-hook.service agentrt-plugin.service
Wants=agentrt-market.service
Wants=agentrt-gateway.service

[Install]
WantedBy=multi-user.target
```

启用 systemd 部署：

```bash
# 安装 service 文件
sudo cp deploy/systemd/*.service /etc/systemd/system/
sudo cp deploy/systemd/agentrt.target /etc/systemd/system/

# 替换占位符
sudo sed -i 's|@CMAKE_INSTALL_PREFIX@|/usr/local|g' /etc/systemd/system/agentrt-*.service
sudo sed -i 's|@AGENTOS_RUNTIME_DIR@|/var/run/agentos|g' /etc/systemd/system/agentrt-*.service

# 重载并启用
sudo systemctl daemon-reload
sudo systemctl enable agentrt.target
sudo systemctl start agentrt.target
```

### 6.3 一键启动脚本

使用 `agentrt-bootstrap.sh` 按 DAG 层级自动启动所有 daemon：

```bash
# 基本启动
bash scripts/ops/agentrt-bootstrap.sh

# 指定二进制目录和配置
bash scripts/ops/agentrt-bootstrap.sh \
  -b ./build/bin \
  -r /var/run/agentos \
  -c /etc/agentos/agentos.yaml \
  -t 120

# Dry-run 模式（仅打印启动计划）
bash scripts/ops/agentrt-bootstrap.sh -n
```

### 6.4 健康检查

每个 daemon 提供 HTTP `/healthz` 端点或 Unix Socket 健康检查。

**Docker 健康检查示例**：

```yaml
# 内核服务
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8091/health"]
  interval: 30s
  timeout: 10s
  retries: 5
  start_period: 60s

# 网关服务
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
  interval: 30s
  timeout: 10s
  retries: 5
  start_period: 45s

# Redis
healthcheck:
  test: ["CMD", "redis-cli", "ping"]
  interval: 10s
  timeout: 5s
  retries: 5

# PostgreSQL
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U agentos -d agentos_prod"]
  interval: 10s
  timeout: 5s
  retries: 5
  start_period: 60s
```

**Kubernetes 健康检查示例**：

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 9001
  initialDelaySeconds: 15
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /healthz
    port: 9001
  initialDelaySeconds: 10
  periodSeconds: 5
```

**daemon 健康检查超时配置**：

| Daemon | 超时（秒） | 检查方式 |
|--------|-----------|---------|
| monit_d, observe_d, info_d, notify_d | 15 | Unix Socket |
| sched_d, channel_d | 20 | Unix Socket |
| llm_d, tool_d | 30 | Unix Socket |
| market_d | 30 | Unix Socket |
| gateway_d | 30 | TCP :8080 |

---

## 7. 数据库设置

### 7.1 PostgreSQL 配置

Airymax 使用 PostgreSQL 15 作为 heapstore 元数据存储。

**Docker Compose 配置**：

```yaml
postgres:
  image: postgres:15-alpine
  container_name: agentos-postgres
  restart: unless-stopped
  environment:
    - POSTGRES_DB=agentos
    - POSTGRES_USER=agentos
    - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
  volumes:
    - postgres-data:/var/lib/postgresql/data
    - ./agentos/heapstore/registry/sessions.db:/docker-entrypoint-initdb.d/sessions.db:ro
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U agentos"]
    interval: 10s
    timeout: 5s
    retries: 5
```

**生产环境 PostgreSQL 性能调优**：

```yaml
postgres-prod:
  command: >
    postgres
    -c max_connections=200
    -c shared_buffers=512MB
    -c effective_cache_size=1GB
    -c maintenance_work_mem=128MB
    -c checkpoint_completion_target=0.9
    -c wal_buffers=16MB
    -c default_statistics_target=100
    -c random_page_cost=1.1
    -c effective_io_concurrency=200
    -c work_mem=4MB
    -c min_wal_size=1GB
    -c max_wal_size=4GB
```

**关键参数说明**：

| 参数 | 值 | 说明 |
|------|-----|------|
| `max_connections` | 200 | 最大连接数 |
| `shared_buffers` | 512MB | 共享缓冲区（建议为内存的 25%） |
| `effective_cache_size` | 1GB | 操作系统缓存估算 |
| `work_mem` | 4MB | 单查询排序内存 |
| `wal_buffers` | 16MB | WAL 缓冲区 |
| `min_wal_size` / `max_wal_size` | 1GB / 4GB | WAL 大小范围 |

### 7.2 数据持久化目录

| 数据卷 | 路径 | 用途 |
|--------|------|------|
| `postgres-data` | `/var/lib/postgresql/data` | PostgreSQL 主数据 |
| `postgres-backups` | `/var/lib/postgresql/backups` | 备份目录 |
| `redis-data` | `/data` | Redis AOF 持久化 |
| `prod-atoms-data` | `/var/lib/agentos` | 内核运行时数据 |
| `prod-daemon-data` | `/var/lib/agentos` | daemon 运行时数据 |
| `jaeger-data` | `/var/lib/jaeger` | Jaeger 追踪数据 |
| `prometheus-data` | `/var/lib/prometheus` | Prometheus 时序数据 |
| `grafana-data` | `/var/lib/grafana` | Grafana 配置数据 |

### 7.3 数据库初始化

数据库初始化通过挂载 SQL 脚本到 `/docker-entrypoint-initdb.d/` 实现：

```yaml
volumes:
  - ./agentos/heapstore/registry/sessions.db:/docker-entrypoint-initdb.d/sessions.db:ro
```

PostgreSQL 容器首次启动时会自动执行该目录下的 `.sql` 和 `.db` 文件。

**手动初始化**：

```bash
# 进入 PostgreSQL 容器
docker-compose exec postgres psql -U agentos -d agentos

# 执行迁移脚本
psql -U agentos -d agentos -f /path/to/migration.sql
```

### 7.4 Redis 配置

Redis 作为缓存和会话存储：

```yaml
redis:
  image: redis:7-alpine
  command: redis-server --appendonly yes

# 生产环境
redis-prod:
  command: >
    redis-server
    --appendonly yes
    --maxmemory 2gb
    --maxmemory-policy allkeys-lru
    --tcp-keepalive 300
    --timeout 0
    --loglevel notice
```

**关键参数说明**：

| 参数 | 值 | 说明 |
|------|-----|------|
| `appendonly` | yes | 启用 AOF 持久化 |
| `maxmemory` | 2gb | 最大内存限制 |
| `maxmemory-policy` | allkeys-lru | 内存淘汰策略（LRU） |
| `tcp-keepalive` | 300 | TCP 保活间隔（秒） |

---

## 8. 监控设置

### 8.1 Prometheus 指标采集

Prometheus 配置文件位于 `deploy/docker/monitoring/prometheus.yml`：

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    monitor: 'agentos-gateway'
    environment: 'production'

rule_files:
  - '/etc/prometheus/rules/*.yml'

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

scrape_configs:
  # Airymax 自身指标
  - job_name: 'agentos-gateway'
    static_configs:
      - targets: ['gateway:9090']
    metrics_path: /metrics

  # Redis 监控
  - job_name: 'redis'
    static_configs:
      - targets: ['redis:6379']

  # Kubernetes Pod 服务发现
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
            - agentos
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

Kubernetes Pod 注解自动发现：

```yaml
podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "9090"
  prometheus.io/path: "/metrics"
```

### 8.2 告警规则

告警规则位于 `deploy/docker/monitoring/alerts.yml`，包含以下告警组：

| 告警组 | 关键告警 | 严重级别 |
|--------|---------|---------|
| `availability` | InstanceDown, HealthCheckFailed | critical / warning |
| `performance` | HighLatency (>1s), HighErrorRate (>5%), HighMemoryUsage, HighCPUUsage | warning |
| `sessions` | TooManySessions (>10000), SessionCreationFailure, SessionCleanupSlow | warning / info |
| `connections` | ConnectionPoolExhausted, TooManyWebSocketConnections (>5000), HighConnectionFailureRate | critical / warning |
| `security` | HighAuthenticationFailure, RateLimitTriggered, JWTTokenExpired | warning / info |
| `dependencies` | RedisConnectionFailed, RedisHighLatency | critical / warning |

示例告警规则：

```yaml
groups:
  - name: agentos-gateway.availability
    interval: 30s
    rules:
      - alert: InstanceDown
        expr: up{job="agentos-gateway"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Gateway 实例宕机"
          description: "实例 {{ $labels.instance }} 已宕机超过 1 分钟"

  - name: agentos-gateway.performance
    interval: 30s
    rules:
      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "HTTP 请求延迟过高"
          description: "95% 分位延迟 {{ $value | humanizeDuration }}，超过 1 秒阈值"
```

### 8.3 Grafana 仪表盘

Grafana 配置文件位于 `deploy/docker/monitoring/grafana_agentrt_dashboard.json`。

**Docker Compose 中的 Grafana 配置**：

```yaml
grafana-prod:
  image: grafana/grafana:10.2.0
  environment:
    - GF_SECURITY_ADMIN_USER=admin
    - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD:?GRAFANA_PASSWORD is required}
    - GF_USERS_ALLOW_SIGN_UP=false
    - GF_ALERTING_ENABLED=true
    - GF_LOG_MODE=console
    - GF_LOG_LEVEL=warn
  volumes:
    - grafana-data:/var/lib/grafana
    - ./ecosystem/manager/grafana/provisioning:/etc/grafana/provisioning:ro
    - ./dashboards:/var/lib/grafana/dashboards:ro
  ports:
    - "127.0.0.1:3000:3000"
```

### 8.4 监控端口汇总

| 服务 | 端口 | 用途 |
|------|------|------|
| Gateway Metrics | 9090 | Daemon 自身指标 |
| monit_d Metrics | 9902 | 平台监控指标 |
| Prometheus | 9090 | 指标采集与查询 |
| Grafana | 3000 | 可视化仪表盘 |
| Jaeger UI | 16686 | 分布式追踪查询 |
| OpenTelemetry Collector | 4317 (gRPC) / 4318 (HTTP) | 遥测数据接收 |
| AlertManager | 9093 | 告警管理 |

---

## 9. 日志与追踪设置

### 9.1 日志配置

Airymax 支持结构化日志（JSON）和普通文本格式：

```bash
# 环境变量
AGENTOS_LOG_LEVEL=INFO      # DEBUG, INFO, WARNING, ERROR, CRITICAL
AGENTOS_LOG_FORMAT=json     # json 或 text
AGENTOS_STRUCTURED_LOGGING=false
```

**开发环境日志**：

```yaml
environment:
  - AGENTOS_LOG_LEVEL=INFO
  - AGENTOS_LOG_FORMAT=text
```

**生产环境日志**：

```yaml
environment:
  - AGENTOS_LOG_LEVEL=WARN
  - AGENTOS_LOG_FORMAT=json
```

### 9.2 Docker 日志驱动

生产环境使用 `json-file` 驱动并配置日志轮转：

```yaml
logging:
  driver: "json-file"
  options:
    max-size: "100m"    # 单文件最大 100 MB
    max-file: "10"      # 最多保留 10 个文件
```

各组件日志文件大小建议：

| 组件 | max-size | max-file | 说明 |
|------|----------|----------|------|
| gateway / atoms / daemon | 100m | 10 | 业务日志量较大 |
| Redis / PostgreSQL | 50m | 5 | 数据库日志 |
| Nginx | 100m | 10 | 访问日志 |
| Prometheus / Grafana | 50m | 5 | 监控系统日志 |

### 9.3 日志持久化

所有 daemon 日志挂载到宿主机目录：

```yaml
volumes:
  # 内核日志
  - ./agentos/heapstore/logs:/var/log/agentos:rw
  # 服务日志
  - ./agentos/heapstore/logs/services:/var/log/agentos/services:rw
```

### 9.4 分布式追踪 (Tracing)

Airymax 使用 OpenTelemetry + Jaeger 实现分布式追踪。

**架构**：

```
Airymax Daemon → OTLP (gRPC :4317) → OpenTelemetry Collector → Jaeger (:14250)
                                                              → Prometheus (:8888)
```

**OpenTelemetry Collector 配置**：

```yaml
otel-collector:
  image: otel/opentelemetry-collector-contrib:0.90.0
  ports:
    - "4317:4317"   # OTLP gRPC
    - "4318:4318"   # OTLP HTTP
    - "8888:8888"   # Prometheus metrics
  volumes:
    - ./ecosystem/manager/otel-collector-manager.yaml:/etc/otelcol-contrib/manager.yaml:ro
  healthcheck:
    test: ["CMD", "wget", "--spider", "http://localhost:13133/"]
    interval: 30s
    timeout: 10s
    retries: 3
```

**Jaeger 配置**：

```yaml
jaeger:
  image: jaegertracing/all-in-one:1.50
  environment:
    - COLLECTOR_OTLP_ENABLED=true
    # 生产环境使用 Badger 持久化
    - SPAN_STORAGE_TYPE=badger
    - BADGER_EPHEMERAL=false
    - BADGER_DIRECTORY_VALUE=/var/lib/jaeger/data
    - BADGER_DIRECTORY_KEY=/var/lib/jaeger/key
  ports:
    - "16686:16686"   # Jaeger UI
    - "14268:14268"   # Thrift HTTP
    - "14250:14250"   # gRPC
  volumes:
    - jaeger-data:/var/lib/jaeger
```

**环境变量启用追踪**：

```bash
AGENTOS_ENABLE_TRACING=true
AGENTOS_TELEMETRY_ENDPOINT=http://otel-collector:4317
AGENTOS_TRACING_SAMPLE_RATE=1.0    # 生产环境建议 0.1
```

---

## 10. 备份与灾难恢复

### 10.1 数据备份

#### PostgreSQL 备份

**自动备份（Docker）**：

```bash
# 使用 Makefile
cd deploy/docker
make backup
```

**手动备份**：

```bash
# 备份 PostgreSQL 数据库
docker exec agentos-postgres-prod pg_dump -U agentos -d agentos_prod \
  > backup_$(date +%Y%m%d_%H%M%S).sql

# 压缩备份
docker exec agentos-postgres-prod pg_dump -U agentos -d agentos_prod \
  | gzip > backup_$(date +%Y%m%d_%H%M%S).sql.gz

# 使用 pg_dumpall 备份全部数据库
docker exec agentos-postgres-prod pg_dumpall -U agentos \
  | gzip > full_backup_$(date +%Y%m%d_%H%M%S).sql.gz
```

**定时备份 Cron Job**：

```bash
# 添加到 crontab（每天凌晨 2 点执行）
0 2 * * * docker exec agentos-postgres-prod pg_dump -U agentos -d agentos_prod \
  | gzip > /backup/agentos_$(date +\%Y\%m\%d).sql.gz

# 保留最近 30 天的备份
0 3 * * * find /backup -name "agentos_*.sql.gz" -mtime +30 -delete
```

#### Redis 备份

```bash
# Redis 已启用 AOF 持久化，数据文件位于 /data/appendonly.aof
# 手动触发 RDB 快照
docker exec agentos-redis-prod redis-cli BGSAVE

# 备份 Redis 数据
docker cp agentos-redis-prod:/data/appendonly.aof ./redis_backup_$(date +%Y%m%d).aof
```

#### 数据卷备份

```bash
# 备份所有数据卷
docker run --rm \
  -v $(docker volume ls -q -f name=agentos | tr '\n' ' ') \
  -v $(pwd)/backup:/backup \
  alpine tar czf /backup/agentos-full-backup-$(date +%Y%m%d-%H%M%S).tar.gz \
  /var/lib/postgresql/data /data
```

### 10.2 灾难恢复

#### PostgreSQL 恢复

```bash
# 1. 停止应用服务（保留数据库）
docker-compose stop agentos-gateway agentos-daemon agentos-atoms

# 2. 恢复数据库
docker exec -i agentos-postgres-prod psql -U agentos -d agentos_prod \
  < backup_20260618_020000.sql

# 或从压缩文件恢复
gunzip -c backup_20260618_020000.sql.gz | \
  docker exec -i agentos-postgres-prod psql -U agentos -d agentos_prod

# 3. 重启应用服务
docker-compose start agentos-atoms agentos-daemon agentos-gateway
```

#### Redis 恢复

```bash
# 停止 Redis
docker-compose stop redis-prod

# 替换 AOF 文件
docker cp redis_backup_20260618.aof agentos-redis-prod:/data/appendonly.aof

# 重启 Redis
docker-compose start redis-prod
```

### 10.3 备份策略建议

| 数据 | 备份频率 | 保留期限 | 方式 |
|------|---------|---------|------|
| PostgreSQL | 每日全量 + 每小时 WAL 归档 | 30 天 | `pg_dump` + WAL archiving |
| Redis | 每日 BGSAVE | 7 天 | AOF + RDB |
| 配置文件 | 每次变更 | 永久 | Git 版本控制 |
| 日志 | 可选 | 30 天 | 日志轮转后归档 |

### 10.4 恢复演练

建议每季度执行一次恢复演练：

```bash
#!/bin/bash
# disaster-recovery-drill.sh

echo "=== 灾难恢复演练 ==="

# 1. 验证备份完整性
echo "[1/4] 验证备份文件..."
ls -lh /backup/agentos_$(date +%Y%m%d).sql.gz

# 2. 在隔离环境恢复
echo "[2/4] 恢复数据库到测试实例..."
gunzip -c /backup/agentos_$(date +%Y%m%d).sql.gz | \
  docker exec -i test-postgres psql -U agentos -d agentos_recovery_test

# 3. 验证数据完整性
echo "[3/4] 验证数据完整性..."
docker exec test-postgres psql -U agentos -d agentos_recovery_test \
  -c "SELECT count(*) FROM sessions;"

# 4. 启动服务冒烟测试
echo "[4/4] 冒烟测试..."
curl -f http://localhost:8080/health

echo "=== 恢复演练完成 ==="
```

---

## 11. 安全加固

### 11.1 TLS / HTTPS

生产环境通过 Nginx 反向代理提供 TLS 终止：

```yaml
nginx-prod:
  image: nginx:1.27-alpine
  ports:
    - "80:80"
    - "443:443"
  volumes:
    - ./ecosystem/manager/nginx/ssl:/etc/nginx/ssl:ro
    - ./ecosystem/manager/nginx/conf.d:/etc/nginx/conf.d:ro
```

Nginx SSL 配置示例：

```nginx
server {
    listen 443 ssl http2;
    server_name agentrt.example.com;

    ssl_certificate     /etc/nginx/ssl/agentrt.crt;
    ssl_certificate_key /etc/nginx/ssl/agentrt.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://agentos-gateway:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Kubernetes Ingress TLS 配置：

```yaml
ingress:
  enabled: true
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: agentrt.example.com
      paths:
        - path: /
          pathType: Prefix
          port: 8080
  tls:
    - secretName: agentrt-tls
      hosts:
        - agentrt.example.com
```

### 11.2 防火墙规则

#### 端口暴露原则

**仅暴露必要的端口**，所有内部服务端口绑定到 `127.0.0.1`：

```yaml
# 正确：仅本地访问
ports:
  - "127.0.0.1:8080:8080"

# 错误：对外暴露
ports:
  - "8080:8080"
```

#### 推荐防火墙规则

```bash
# 使用 iptables 限制入站流量
# 仅允许 80/443 (HTTP/HTTPS)
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# 允许内部服务端口（仅 localhost）
iptables -A INPUT -p tcp -s 127.0.0.1 --dport 8080 -j ACCEPT
iptables -A INPUT -p tcp -s 127.0.0.1 --dport 9090 -j ACCEPT

# 拒绝所有其他入站连接
iptables -A INPUT -j DROP
```

#### 网络隔离（Docker）

生产环境使用双网络架构：

```yaml
networks:
  agentos-production-network:   # 外部网络（Nginx + Gateway）
    driver: bridge
    ipam:
      config:
        - subnet: 172.30.0.0/16

  agentos-internal-network:     # 内部网络（daemon + 数据库）
    driver: bridge
    internal: false
    ipam:
      config:
        - subnet: 172.31.0.0/16
```

### 11.3 容器安全

生产环境所有容器启用安全加固：

```yaml
security_opt:
  - no-new-privileges:true    # 禁止提权
cap_drop:
  - ALL                       # 移除所有 capabilities
read_only: true               # 只读根文件系统（如适用）

# Nginx 例外：需要绑定特权端口
cap_add:
  - NET_BIND_SERVICE
```

**非 root 用户运行**（Dockerfile）：

```dockerfile
RUN addgroup -g 1000 agentos && \
    adduser -u 1000 -G agentos -s /bin/sh -D agentos

USER agentos
```

**Kubernetes SecurityContext**：

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  capabilities:
    drop:
      - ALL
```

### 11.4 Secrets 管理

**Docker 环境**：

```bash
# .env 文件不提交到 Git
echo ".env" >> .gitignore

# 密钥文件放在 secrets/ 目录
deploy/docker/secrets/
└── .gitkeep    # 实际密钥文件在此目录下，不提交 Git
```

**Kubernetes 环境**：

```bash
# 创建 PostgreSQL 密码 Secret
kubectl create secret generic agentrt-postgresql-secret \
  --from-literal=password='<generated-password>' \
  -n agentrt

# 创建 API 密钥 Secret
kubectl create secret generic agentrt-api-keys \
  --from-literal=openai-api-key='sk-...' \
  -n agentrt

# 使用外部 Secrets 管理工具（推荐）
# - HashiCorp Vault
# - AWS Secrets Manager
# - Azure Key Vault
# - Sealed Secrets
```

### 11.5 JWT 认证

```bash
# 生成安全的 JWT 密钥
openssl rand -hex 32

# 设置环境变量
AGENTOS_JWT_SECRET=<generated-secret>
```

### 11.6 速率限制

```bash
# API 速率限制（每分钟请求数）
API_RATE_LIMIT=100

# CORS 限制（生产环境必须指定具体域名）
ENABLE_CORS=true
CORS_ALLOWED_ORIGINS=https://your-frontend-domain.com
```

### 11.7 镜像安全

```bash
# 定期扫描镜像漏洞
docker scan ghcr.io/spharx/agentos-gateway:0.1.0

# 使用 Trivy 扫描
trivy image ghcr.io/spharx/agentos-gateway:0.1.0

# 最小化基础镜像
# 使用 Alpine 而非 Ubuntu（减小攻击面）
FROM alpine:3.19 AS runtime
```

---

## 12. 生产环境检查清单

### 12.1 部署前检查

- [ ] 所有 API 密钥已配置且非默认值
- [ ] `POSTGRES_PASSWORD` 已修改为强密码
- [ ] `AGENTOS_JWT_SECRET` 已生成并配置
- [ ] `.env` 文件已排除出 Git 版本控制
- [ ] 运行 `check_config.sh` 通过所有检查项
- [ ] 端口 80/443/8080 等无冲突
- [ ] 磁盘空间充足（至少 200 GB 可用）
- [ ] Docker Engine ≥ 20.10
- [ ] 系统时间已同步（NTP）

### 12.2 配置检查

- [ ] 日志级别设为 `WARN`（生产环境）
- [ ] 日志格式设为 `json`（便于日志聚合）
- [ ] 所有容器设置了资源限制（CPU / Memory）
- [ ] 所有容器配置了健康检查
- [ ] 所有服务端口绑定到 `127.0.0.1`（仅 Nginx 对外暴露）
- [ ] `CORS_ALLOWED_ORIGINS` 限制为具体域名
- [ ] 追踪采样率设为 `0.1`（生产环境）
- [ ] 日志轮转已配置（max-size, max-file）
- [ ] PostgreSQL 生产参数已优化（shared_buffers, max_connections 等）
- [ ] Redis maxmemory 和淘汰策略已配置

### 12.3 安全检查

- [ ] 所有容器启用 `no-new-privileges:true`
- [ ] 所有容器移除不必要 capabilities（`cap_drop: ALL`）
- [ ] 容器以非 root 用户运行（UID 1000）
- [ ] TLS 证书已配置且有效
- [ ] 防火墙规则已配置（仅开放必要端口）
- [ ] Secrets 通过 K8s Secret 或 Docker Secrets 管理
- [ ] 镜像已通过安全扫描
- [ ] 基础镜像使用最新稳定版 Alpine
- [ ] 数据库不对外暴露（仅内部网络访问）

### 12.4 监控检查

- [ ] Prometheus 指标采集已配置
- [ ] 告警规则已部署并验证
- [ ] Grafana 仪表盘已导入
- [ ] AlertManager 通知渠道已配置（邮件 / Slack / PagerDuty）
- [ ] Jaeger 追踪已启用
- [ ] OpenTelemetry Collector 运行正常
- [ ] 健康检查端点可访问（`/healthz`）

### 12.5 备份检查

- [ ] PostgreSQL 定时备份已配置（每天）
- [ ] Redis AOF 持久化已启用
- [ ] 备份文件存储到独立存储（非本地磁盘）
- [ ] 备份保留策略已配置（至少 30 天）
- [ ] 恢复演练计划已制定（每季度一次）

### 12.6 高可用检查

- [ ] 关键 daemon 至少 2 个副本（gateway 3 个）
- [ ] Pod 反亲和性已配置（Kubernetes）
- [ ] PostgreSQL 有只读副本（生产环境）
- [ ] Redis 使用主从模式（生产环境）
- [ ] 自动重启策略已配置（`restart: always` / `on-failure`）

### 12.7 部署后验证

```bash
# 1. 检查所有容器运行状态
docker-compose -f docker-compose.prod.yml ps
# 或
kubectl get pods -n agentrt

# 2. 验证健康检查
curl -f http://localhost:8080/health
curl -f http://localhost:8091/health

# 3. 验证指标端点
curl http://localhost:9090/metrics

# 4. 验证追踪
curl http://localhost:16686/api/services

# 5. 验证数据库连接
docker exec agentos-postgres-prod pg_isready -U agentos -d agentos_prod

# 6. 验证 Redis 连接
docker exec agentos-redis-prod redis-cli ping

# 7. 测试基本 API 请求
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <your-api-key>" \
  -d '{"model": "gpt-4", "messages": [{"role": "user", "content": "Hello"}]}'

# 8. 查看日志
docker-compose -f docker-compose.prod.yml logs --tail=50 gateway
```

---

## 附录

### A. 快速参考

| 操作 | 命令 |
|------|------|
| 快速启动 | `cd deploy/docker && ./quickstart.sh` |
| 配置检查 | `cd deploy/docker && ./check_config.sh` |
| 构建镜像 | `cd deploy/docker && make build-all` |
| 启动服务 | `cd deploy/docker && make start` |
| 查看状态 | `cd deploy/docker && make ps` |
| 查看日志 | `cd deploy/docker && make logs` |
| 健康检查 | `cd deploy/docker && make health` |
| 备份数据 | `cd deploy/docker && make backup` |
| K8s 安装 | `helm install agentrt ./deploy/kubernetes/helm -n agentrt --create-namespace` |
| K8s 升级 | `helm upgrade agentrt ./deploy/kubernetes/helm -f ./deploy/kubernetes/helm/values-prod.yaml -n agentrt` |

### B. 端口总览

| 端口 | 协议 | 服务 | 对外 |
|------|------|------|------|
| 80 | HTTP | Nginx | 是 |
| 443 | HTTPS | Nginx (TLS) | 是 |
| 8080 | HTTP | Gateway | 否（127.0.0.1） |
| 8081 | WebSocket | Gateway | 否（127.0.0.1） |
| 8443 | HTTPS | Gateway (TLS) | 否（127.0.0.1） |
| 9090 | HTTP | Gateway Metrics | 否（127.0.0.1） |
| 3000 | HTTP | Grafana | 否（127.0.0.1） |
| 9090 | HTTP | Prometheus | 否（127.0.0.1） |
| 16686 | HTTP | Jaeger UI | 否（127.0.0.1） |
| 4317 | gRPC | OTLP Collector | 否（127.0.0.1） |
| 5432 | TCP | PostgreSQL | 否（内部网络） |
| 6379 | TCP | Redis | 否（内部网络） |

### C. 相关文档

- `deploy/README.md` — 部署目录总览
- `deploy/docker/README.md` — Docker 部署详细文档
- `deploy/kubernetes/README.md` — Kubernetes 部署详细文档
- `scripts/ops/README.md` — 运维脚本总览

---

> © 2025-2026 SPHARX Ltd. All Rights Reserved.  
> "From data intelligence emerges."