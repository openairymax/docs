# AgentOS Docker 部署完整指南

**最新**: 2026-06-09
**状态**: 维护中
**路径**: OpenAirymax/Docs/Capital_API/docker/README.md
**作者**:
    - Liren Wang
---

## 🎯 三种部署方式

### ⚡ 方式1: Docker Compose (推荐用于生产环境)

**适用场景**: 完整的多服务部署，包含网关、LLM、监控等

```bash
# 1. 进入docker目录
cd Docs/Capital_API/docker

# 2. 配置环境变量（可选）
cp .env.example .env
vim .env  # 编辑你的API密钥等配置

# 3. 一键启动所有服务
docker-compose up -d

# 4. 查看状态
docker-compose ps
docker-compose logs -f
```

**访问地址**:
- HTTP API: http://localhost:8080
- WebSocket: ws://localhost:8081
- Metrics: http://localhost:9090/metrics
- Market API: http://localhost:9000

---

### 🔧 方式2: 单容器运行 (推荐用于开发/测试)

**适用场景**: 快速测试单个服务，或嵌入到现有应用中

#### 运行网关服务

```bash
# 构建镜像
docker build -t agentos:latest .

# 基本运行
docker run -d \
  --name agentos-gateway \
  -p 8080:8080 \
  -p 8081:8081 \
  agentos:latest

# 带自定义配置运行
docker run -d \
  --name agentos-gateway \
  -p 9090:8080 \
  -v $(pwd)/config:/etc/agentos:ro \
  -e AGENTOS_LOG_LEVEL=debug \
  -e OPENAI_API_KEY=sk-your-key-here \
  agentos:latest gateway_d -c /etc/agentos/gateway.conf
```

#### 进入容器调试

```bash
# 交互式shell
docker run -it --rm agentos:latest bash

# 查看日志
docker logs -f agentos-gateway

# 执行命令
docker exec agentos-gateway gateway_d --help
```

---

### 🛠️ 方式3: 开发环境 (挂载源码)

**适用场景**: 本地开发，代码修改后自动重新编译

```bash
# 挂载源码目录
docker run -it --rm \
  -v /path/to/AgentOS:/workspace/AgentOS \
  -w /workspace/AgentOS/build \
  agentos:latest bash

# 在容器内执行构建命令
cmake .. -DCMAKE_BUILD_TYPE=Debug
make -j$(nproc)
ctest --output-on-failure
```

**使用docker-compose开发模式**:

```yaml
# docker-compose.dev.yml (用于开发)
version: '3.8'
services:
  dev:
    build:
      context: ../../
      dockerfile: Docs/Capital_API/docker/Dockerfile.dev
    volumes:
      - ../..:/workspace/AgentOS
    working_dir: /workspace/AgentOS/build
    command: bash -c "cmake .. && make -j$(nproc) && ctest"
```

---

## 📦 镜像说明

### 镜像标签 (Tags)

| 标签 | 说明 | 大小 |
|------|------|------|
| `v0.0.4` | 最新稳定版 | ~500MB |
| `latest` | 同v0.0.4 | ~500MB |
| `v0.0.4-alpine` | Alpine轻量版 | ~150MB |
| `dev` | 开发版(含调试工具) | ~800MB |

### 镜像内容

```
/usr/local/
├── bin/
│   ├── gateway_d          # 网关守护进程
│   ├── agentos-llm-d      # LLM服务
│   ├── channel_d          # 通道服务
│   ├── agentos-sched-d    # 调度器
│   ├── agentos-monit-d    # 监控服务
│   ├── agentos-market-d   # 市场服务
│   └── agentos-tool-d     # 工具执行器
├── lib/
│   ├── libagentos_common.a
│   ├── libagentos_core.a
│   ├── libagentos_protocols.so → libagentos_protocols.so.2.1.0
│   └── ... (其他库文件)
├── include/
│   └── agentos.h          # 主头文件
└── share/doc/agentos/
    └── README.md
```

---

## 🔐 安全最佳实践

### 1. 最小权限原则

```bash
# 使用非root用户运行（镜像已内置agentos用户）
# Dockerfile中已配置: USER agentos

# 限制Linux capabilities
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE agentos:latest

# 只读文件系统（必要时挂载可写卷）
docker run --read-only \
  -v agentos-data:/var/lib/agentos/data \
  -v agentos-logs:/var/log/agentos \
  agentos:latest
```

### 2. 网络隔离

```bash
# 创建独立网络
docker network create agentos-net --subnet=172.28.0.0/16

# 只暴露必要端口
docker run --network agentos-net \
  -p 127.0.0.1:8080:8080 \  # 仅绑定localhost
  agentos:latest

# 内部网络（不暴露端口）
docker run --network agentos-net \
  --expose 8080 \
  agentos:latest
```

### 3. 资源限制

```bash
# 限制内存和CPU
docker run -m 512m --cpus="1.5" agentos:latest

# 在docker-compose.yml中配置:
services:
  gateway:
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '1.5'
        reservations:
          memory: 256M
          cpus: '0.5'
```

### 4. 密钥管理

**❌ 错误做法**: 将API密钥写入docker-compose.yml

```yaml
# 不要这样做!
environment:
  - OPENAI_API_KEY=sk-abc123...
```

**✅ 正确做法**: 使用secrets或.env文件

```bash
# 方式A: .env文件
echo "OPENAI_API_KEY=sk-xxx" > .env
docker-compose --env-file .env up -d

# 方式B: Docker secrets (Swarm mode)
echo "sk-xxx" | docker secret create openai_api_key -
# 在compose中使用:
environment:
  - OPENAI_API_KEY_FILE=/run/secrets/openai_api_key
secrets:
  - openai_api_key
```

---

## 📊 监控与日志

### 日志管理

```bash
# 查看实时日志
docker-compose logs -f gateway

# 查看最近100行
docker-compose logs --tail=100 llm-service

# 导出日志到文件
docker-compose logs > agentos_$(date +%Y%m%d).log 2>&1

# 结构化日志 (JSON格式)
docker run --log-driver=json-file \
  --log-opt labels="service=gateway" \
  agentos:latest
```

### 健康检查

```bash
# 手动健康检查
curl http://localhost:8080/health

# 预期响应:
{
  "status": "ok",
  "service": "gateway",
  "version": "0.1.0",
  "uptime_seconds": 3600,
  "requests_total": 1234
}

# Docker内置健康检查
docker inspect --format='{{.State.Health.Status}}' agentos-gateway
# 输出: healthy / unhealthy / starting
```

### Prometheus指标集成

```bash
# 访问metrics端点
curl http://localhost:9090/metrics

# 输出示例:
# HELP agentos_requests_total Total requests processed
TYPE agentos_requests_total counter
agentos_requests_total{service="gateway",endpoint="/api/v1/chat"} 567.000

# HELP agentos_request_duration_seconds Request duration in seconds
TYPE agentos_request_duration_seconds histogram
agentos_request_duration_seconds_bucket{le="0.1"} 100.000
agentos_request_duration_seconds_bucket{le="0.5"} 250.000
agentos_request_duration_seconds_bucket{le="1.0"} 300.000
agentos_request_duration_seconds_sum 125.500
agentos_request_duration_seconds_count 300.000
```

**Grafana Dashboard配置**:

导入JSON (在Grafana UI → Import):
```json
{
  "dashboard": {
    "title": "AgentOS Monitoring",
    "panels": [...]
  },
  "datasource": {
    "type": "prometheus",
    "url": "http://monitor:9090"
  }
}
```

---

## 🔄 升级策略

### 滚动升级 (Zero Downtime)

```bash
# 1. 拉取新版本
docker pull spharx/agentos:v0.1.0

# 2. 更新镜像标签
sed -i 's/image: spharx\/agentos:.*/image: spharx\/agentos:v0.1.0/' docker-compose.yml

# 3. 逐个重启服务
docker-compose up -d --no-deps gateway     # 先重启gateway
sleep 10                                 # 等待健康检查通过
docker-compose up -d --no-deps llm-service
# ... 其他服务
```

### 数据迁移

```bash
# 备份当前数据
docker run --rm \
  -v agentos-data:/data \
  -v $(pwd)/backup:/backup \
  alpine tar czf /backup/backup-$(date +%Y%m%d-%H%M%S).tar.gz -C /data .

# 恢复到新版本
docker run --rm \
  -v agentos-new-data:/data \
  -v $(pwd)/backup:/backup \
  alpine sh -c "tar xzf /backup/backup-latest.tar.gz -C /data"

# 验证数据完整性
docker exec agentos-gateway ls -la /var/lib/agentos/data/
```

### 回滚操作

```bash
# 回滚到上一版本
git checkout HEAD~1 docker-compose.yml  # 或手动编辑
docker-compose down
docker-compose up -d

# 或使用特定镜像标签
docker tag spharx/agentos:v0.0.3 spharx/agentos:rollback
docker-compose up -d
```

---

## 🐛 故障排查

### 容器无法启动

```bash
# 检查日志
docker logs agentos-gateway

# 常见原因及解决:

# 1. 端口被占用
# 解决: 修改映射端口 -p 9090:8080

# 2. 权限不足
# 解决: 检查数据目录权限
ls -la /var/lib/agentos/

# 3. 配置文件错误
# 解决: 进入容器检查配置
docker exec -it agentos-gateway cat /etc/agentos/gateway.conf

# 4. 内存不足
# 解决: 增加内存限制 -m 1g
```

### 性能问题

```bash
# 查看资源使用
docker stats agentos-gateway

# 分析CPU使用
docker exec agentos-gateway top

# 如果内存持续增长，可能是内存泄漏
# 尝试重启并观察:
docker restart agentos-gateway
docker stats --no-stream agentos-gateway
```

### 网络问题

```bash
# 测试容器间连通性
docker exec agentos-gateway ping llm-service

# 检查端口绑定
docker port agentos-gateway

# 查看网络详情
docker network inspect bridge
```

---

## 📈 性能优化建议

### 镜像优化

```dockerfile
# 多阶段构建减少最终镜像大小
FROM ubuntu:22.04 AS builder
# ... 编译步骤 ...

FROM ubuntu:22.04 AS runtime
COPY --from=builder /usr/local/bin/* /usr/local/bin/
COPY --from=builder /usr/local/lib/*.so* /usr/local/lib/
# 不复制头文件、文档、调试符号
```

### 运行时优化

```yaml
# docker-compose.yml优化配置
services:
  gateway:
    environment:
      # 减少日志输出
      - AGENTOS_LOG_LEVEL=warn
      
      # 调整线程池大小
      - AGENTOS_EXECUTION_THREADS=8
      
      # 启用缓存
      - AGENTOS_LLM_CACHE_ENABLED=true
      - AGENTOS_LLM_CACHE_SIZE=2000
    
    # 使用tmpfs减少磁盘I/O
    tmpfs:
      - /tmp:size=64M,mode=1777
      - /var/run/agentos:size=16M,mode=1777
```

---

## 🌍 生产部署清单

### 部署前检查

- [ ] 所有敏感信息使用secrets/.env管理
- [ ] 网络策略已配置（仅开放必要端口）
- [ ] 资源限制已设置（memory/cpu）
- [ ] 日志驱动已配置（json-file/syslog）
- [ ] 健康检查端点可访问
- [ ] 数据卷已正确挂载并备份
- [ ] 监控系统已集成（Prometheus/Grafana）
- [ ] 回滚方案已测试

### 部署后验证

```bash
# 1. 检查所有容器状态
docker-compose ps
# 应显示所有服务为 "Up" 状态

# 2. 健康检查
for svc in gateway llm scheduler monitor; do
  curl -sf http://localhost:8080/health && echo "$svc: ✅" || echo "$svc: ❌"
done

# 3. 功能测试
curl -X POST http://localhost:8080/api/v1/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"hello"}'

# 4. 查看初始日志确认无错误
docker-compose logs --tail=50
```

---

## 📞 技术支持

- **Docker Hub**: https://hub.docker.com/r/spharx/agentos
- **GitHub Issues**: https://github.com/spharx/AgentOS/issues?q=is%3Aissue+is%3Aopen+label%3Adocker
- **文档**: https://docs.agentos.dev/docker

---

**最后更新**: 2026-04-28  
**维护者**: SPHARX DevOps Team
