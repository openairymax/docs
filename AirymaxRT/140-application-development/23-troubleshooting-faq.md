# Airymax 故障排查与FAQ手册

**最新**: 2026-06-09
**状态**: 维护中
**路径**: docs/AirymaxRT/140-application-development/23-troubleshooting-faq.md

## 概述

本手册涵盖 Airymax 常见故障的诊断方法、解决方案和预防措施，按系统模块分类组织。

---

## 一、网关故障

### 1.1 网关启动失败

**症状**: `gateway_d` 进程无法启动或立即退出

**诊断步骤**:

```bash
# 1. 检查端口占用
lsof -i :8080 || netstat -tlnp | grep 8080

# 2. 检查配置文件
cat agentrt/gateway/config/gateway.yaml

# 3. 前台模式启动查看详细日志
./bin/gateway_d -h 0.0.0.0 -p 8080 -d

# 4. 检查依赖服务
curl -s http://localhost:6379/ping 2>/dev/null || echo "Redis不可达"
psql -h localhost -U agentrt -c "SELECT 1" 2>/dev/null || echo "PostgreSQL不可达"
```

**常见原因与解决**:

| 原因 | 错误信息 | 解决方案 |
|------|---------|---------|
| 端口被占用 | `bind: address already in use` | 更换端口或终止占用进程 |
| 配置文件缺失 | `config not found` | 检查配置路径，使用 `--config` 指定 |
| Redis不可达 | `redis connection refused` | 启动Redis: `redis-server` |
| 证书无效 | `SSL certificate error` | 检查证书路径和有效期 |
| 权限不足 | `permission denied` | 检查文件权限和运行用户 |

### 1.2 网关响应超时

**症状**: 请求发出后长时间无响应

**诊断步骤**:

```bash
# 1. 检查网关健康状态
curl -s http://localhost:8080/health | python3 -m json.tool

# 2. 检查熔断器状态
curl -s http://localhost:8080/ \
  -d '{"jsonrpc":"2.0","method":"circuit_breaker.status","id":1}'

# 3. 检查服务发现状态
curl -s -X POST http://localhost:8080/ \
  -d '{"jsonrpc":"2.0","method":"service.list","id":1}'

# 4. 检查系统资源
top -bn1 | head -20
df -h
free -h
```

**常见原因与解决**:

| 原因 | 诊断特征 | 解决方案 |
|------|---------|---------|
| 熔断器开启 | `circuit_breaker.status` 返回 OPEN | 等待恢复或检查下游服务 |
| 后端服务不可达 | `service.list` 显示 unhealthy | 重启后端服务 |
| 内存不足 | `free -h` 显示可用内存低 | 增加内存或减少并发 |
| 连接池耗尽 | 大量请求排队 | 增大连接池配置 |
| 死锁 | CPU低但无响应 | 重启网关，检查锁使用 |

---

## 二、协议兼容性故障

### 2.1 JSON-RPC 请求返回 Parse Error

**错误码**: -32700

**诊断**:

```bash
# 验证JSON格式
echo '{"jsonrpc":"2.0","method":"agent.list","id":1}' | python3 -m json.tool

# 常见格式问题
# ❌ 缺少jsonrpc字段
# ❌ id不是数字或字符串
# ❌ params不是对象或数组
# ❌ method不是字符串
```

**解决**: 确保请求严格遵循 JSON-RPC 2.0 规范

### 2.2 MCP 工具调用失败

**症状**: `tools/call` 返回错误

**诊断步骤**:

```bash
# 1. 验证工具是否存在
curl -s -X POST http://localhost:8080/mcp \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}'

# 2. 检查参数格式
# 确保arguments匹配inputSchema定义

# 3. 检查工具服务状态
curl -s -X POST http://localhost:8080/ \
  -d '{"jsonrpc":"2.0","method":"service.list","params":{"protocol":"mcp"},"id":1}'
```

**常见原因**:

| 原因 | 解决方案 |
|------|---------|
| 工具名拼写错误 | 对照 `tools/list` 返回的名称 |
| 参数类型不匹配 | 检查 `inputSchema` 定义 |
| 工具服务离线 | 重启 tool_d 用户态服务 |
| 权限不足 | 检查API Key和权限配置 |

### 2.3 A2A 智能体发现无结果

**症状**: `agent/discover` 返回空列表

**诊断步骤**:

```bash
# 1. 检查是否有注册的智能体
curl -s -X POST http://localhost:8080/ \
  -d '{"jsonrpc":"2.0","method":"agent.list","id":1}'

# 2. 检查服务发现健康
curl -s http://localhost:8080/ \
  -d '{"jsonrpc":"2.0","method":"service.discovery.status","id":1}'

# 3. 检查Redis连接（服务发现依赖Redis）
redis-cli ping
```

### 2.4 OpenAI API 兼容性问题

**症状**: Chat Completions 返回非预期格式

**诊断步骤**:

```bash
# 1. 验证请求格式
curl -v -X POST http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer test-key" \
  -d '{"model":"agentrt-default","messages":[{"role":"user","content":"test"}]}'

# 2. 检查模型列表
curl -s http://localhost:8080/v1/models \
  -H "Authorization: Bearer test-key"

# 3. 检查LLM服务状态
curl -s -X POST http://localhost:8080/ \
  -d '{"jsonrpc":"2.0","method":"service.list","params":{"type":"llm"},"id":1}'
```

**常见原因**:

| 原因 | 解决方案 |
|------|---------|
| model参数错误 | 使用 `/v1/models` 查看可用模型 |
| 缺少Authorization头 | 添加 `Authorization: Bearer <key>` |
| messages格式错误 | 确保role为system/user/assistant |
| LLM服务不可达 | 检查llm_d用户态服务状态 |

---

## 三、IPC与服务发现故障

### 3.1 IPC通信失败

**症状**: 服务间消息传递失败

**诊断步骤**:

```bash
# 1. 检查IPC通道状态
./scripts/agentrt-debug ipc

# 2. 检查共享内存段
ipcs -m

# 3. 检查信号量
ipcs -s

# 4. 检查消息队列
ipcs -q
```

**常见原因**:

| 原因 | 解决方案 |
|------|---------|
| 共享内存不足 | 增大 `/proc/sys/kernel/shmmax` |
| 通道未创建 | 确保发送方先启动 |
| 权限不匹配 | 检查IPC对象权限 |
| 系统限制 | 调整 `/proc/sys/kernel/msgmni` |

### 3.2 服务发现延迟高

**症状**: 服务注册/发现响应慢

**诊断**:

```bash
# 检查Redis延迟
redis-cli --latency

# 检查服务发现配置
cat agentrt/daemons/common/include/service_discovery.h | grep DEFAULT

# 检查心跳间隔
grep heartbeat agentrt/gateway/config/*.yaml
```

**优化方案**:

| 方案 | 配置 |
|------|------|
| 缩短心跳间隔 | `health_check_interval_ms: 3000` |
| 使用本地缓存 | 启用 `svc_cache.h` |
| 调整负载均衡策略 | 使用 `LEAST_CONNECTION` |

---

## 四、熔断器故障

### 4.1 熔断器频繁开启

**症状**: 服务频繁被熔断

**诊断步骤**:

```bash
# 1. 查看熔断器状态
curl -s -X POST http://localhost:8080/ \
  -d '{"jsonrpc":"2.0","method":"circuit_breaker.list","id":1}'

# 2. 查看失败原因
# 检查下游服务日志

# 3. 检查网络连通性
ping downstream-service
traceroute downstream-service
```

**调整建议**:

| 参数 | 默认值 | 建议调整 |
|------|--------|---------|
| `failure_threshold` | 5 | 提高到10（容忍更多瞬态故障） |
| `recovery_timeout_ms` | 30000 | 缩短到15000（更快恢复） |
| `half_open_max_calls` | 3 | 提高到5（更多试探请求） |

### 4.2 熔断器无法恢复

**症状**: 熔断器一直处于OPEN状态

**诊断**:

```bash
# 检查下游服务是否恢复
curl -s http://downstream-service:port/health

# 手动重置熔断器（仅紧急情况）
curl -s -X POST http://localhost:8080/ \
  -d '{"jsonrpc":"2.0","method":"circuit_breaker.reset","params":{"service":"target"},"id":1}'
```

---

## 五、构建与编译故障

### 5.1 CMake配置失败

**常见错误**:

| 错误 | 原因 | 解决方案 |
|------|------|---------|
| `Could not find cJSON` | 缺少cJSON库 | `apt install libcjson-dev` |
| `Could not find libcurl` | 缺少curl库 | `apt install libcurl4-openssl-dev` |
| `Could not find libyaml` | 缺少yaml库 | `apt install libyaml-dev` |
| `CMake version too old` | CMake版本低 | 升级到3.20+ |
| `C compiler too old` | GCC版本低 | 升级到GCC 11+ |

### 5.2 编译错误

| 错误 | 原因 | 解决方案 |
|------|------|---------|
| `implicit declaration` | 缺少头文件包含 | 添加 `#include` |
| `redefinition of struct` | 重复定义 | 使用 include guards |
| `undefined reference` | 链接顺序错误 | 调整CMakeLists.txt链接顺序 |
| `pthread_create` 未定义 | 缺少pthread链接 | 添加 `-lpthread` |

---

## 六、Docker故障

### 6.1 容器启动失败

```bash
# 查看容器日志
docker compose -f docker-compose.dev.yml logs gateway

# 查看容器状态
docker compose -f docker-compose.dev.yml ps

# 进入容器调试
docker compose -f docker-compose.dev.yml exec gateway /bin/sh
```

### 6.2 服务间通信失败

```bash
# 检查Docker网络
docker network ls
docker network inspect airy_default

# 检查DNS解析
docker compose exec gateway nslookup redis
docker compose exec gateway nslookup postgres
```

---

## 七、监控与可观测性故障

### 7.1 Prometheus指标不可用

```bash
# 检查指标端点
curl -s http://localhost:8080/metrics | head -20

# 检查Prometheus配置
cat agentrt/gateway/docker/monitoring/prometheus.yml

# 检查Prometheus目标状态
# 访问 http://localhost:9090/targets
```

### 7.2 Grafana仪表盘无数据

```bash
# 检查数据源配置
# 访问 http://localhost:3000/datasources

# 检查Prometheus是否正常采集
curl -s 'http://localhost:9090/api/v1/query?query=up'

# 重新导入仪表盘
# 使用 agentrt/gateway/docker/monitoring/grafana_airy_dashboard.json
```

---

## 八、FAQ

### Q1: Airymax 支持哪些操作系统？
**A**: Ubuntu 22.04+, macOS 13+, Windows 11 (WSL2)。生产环境推荐Ubuntu 22.04 LTS。

### Q2: 最低硬件要求是什么？
**A**: 开发环境 2核CPU/4GB内存/20GB磁盘；生产环境 4核CPU/8GB内存/50GB磁盘。

### Q3: 如何查看Airymax版本？
**A**: `./gateway_d -v` 或 `curl http://localhost:8080/health`

### Q4: 如何启用多协议支持？
**A**: 编译时添加 `-DENABLE_MCP=ON -DENABLE_A2A=ON -DENABLE_OPENAI=ON`

### Q5: 熔断器的默认超时时间是多少？
**A**: 30秒 (`recovery_timeout_ms = 30000`)，可通过配置文件调整。

### Q6: 如何配置API Key认证？
**A**: 在配置文件中设置 `gateway.auth.enabled: true` 和 `gateway.auth.keys` 列表。

### Q7: 协议转换有性能开销吗？
**A**: 协议转换延迟 < 2ms (P95)，对整体请求延迟影响可忽略。

### Q8: 如何扩展自定义协议？
**A**: 实现 `protocol_handler_t` 接口并注册到协议路由器。参考 `gateway_protocol_handler.h`。

### Q9: 服务发现支持哪些负载均衡策略？
**A**: 5种：Round-Robin、Weighted、Least-Connection、Random、Least-Load。

### Q10: 如何运行诊断工具？
**A**: `./scripts/agentrt-debug` 支持网关/协议/服务/IPC/熔断/告警/配置7类检查。

### Q11: 如何贡献代码？
**A**: 参见 [CONTRIBUTING.md](../../CONTRIBUTING.md)，遵循Conventional Commits规范。

### Q12: 如何报告安全漏洞？
**A**: 发送邮件至 team@spharx.cn，不要公开提交Issue。

---

## 九、诊断工具速查

| 工具 | 路径 | 用途 |
|------|------|------|
| agentrt CLI | `scripts/agentrt` | 统一命令行管理 |
| agentrt-debug | `scripts/agentrt-debug` | 7类诊断检查 |
| 协议测试 | `tests/integration/test_protocol_compatibility.py` | 10类协议测试 |
| 性能基准 | `tests/benchmarks/benchmark_performance.py` | 8项基准测试 |
| 一键发布 | `scripts/release.sh` | 7道质量门禁 |
| 文档一致性 | `scripts/tools/docs/verify_consistency.py` | 文档质量检查 |

---

## 十、安装与启动问题

### Q1: Docker容器启动失败，提示端口被占用？

**症状**:
```
Error starting userland proxy: listen tcp4 0.0.0.0:8080: bind: address already in use
```

**原因**: 8080端口已被其他进程占用。

**解决方案**:

```bash
# 1. 查找占用端口的进程
lsof -i :8080
# 或
ss -tlnp | grep 8080

# 2. 查看进程详情
ps -p <PID> -o pid,cmd

# 3. 选择以下任一方式：
# 方式A: 停止占用进程（如果不重要）
kill <PID>

# 方式B: 修改Airymax端口
# 编辑 docker/.env
KERNEL_IPC_PORT=8081  # 改为其他端口

# 方式C: 停止其他服务后重启Airymax
docker compose -f docker/docker-compose.yml down
docker compose -f docker/docker-compose.yml up -d
```

**预防措施**:

在 `.env` 中自定义端口以避免冲突：
```env
KERNEL_IPC_PORT=18080
POSTGRES_PORT=15432
REDIS_PORT=16379
```

---

### Q2: Docker构建失败，提示内存不足？

**症状**:
```
fatal error: killed signal cc1plus (out of memory)
```

**原因**: 系统可用内存不足以进行并行编译。

**解决方案**:

```bash
# 方式1: 减少Docker构建使用的内存
# 编辑 /etc/docker/daemon.json
{
  "memory": 4096000000,  // 限制Docker使用4GB
  "cpus": 4
}

# 重启Docker
sudo systemctl restart docker

# 方式2: 限制构建并行度
# 编辑 deploy/docker/Dockerfile
RUN cmake --build . --parallel 2  # 从$(nproc)改为固定值

# 方式3: 增加swap空间
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

---

### Q3: 数据库连接失败？

**症状**:
```
could not connect to server: Connection refused
```

**排查步骤**:

```bash
# 1. 检查PostgreSQL容器状态
docker ps | grep postgres

# 2. 查看PostgreSQL日志
docker logs agentrt-postgres-dev 2>&1 | tail -50

# 3. 手动测试连接
docker exec -it agentrt-postgres-dev psql -U agentrt -d agentrt -c "SELECT version();"

# 4. 检查网络连通性
docker exec agentrt-kernel-dev ping postgres -c 3

# 5. 检查环境变量中的连接字符串
grep DATABASE_URL docker/.env
```

**常见原因及解决**:

| 原因 | 解决方案 |
|------|---------|
| PostgreSQL未完全启动 | 等待健康检查通过（约10-30秒） |
| 密码错误 | 检查 `.env` 中的 `POSTGRES_PASSWORD` |
| 网络隔离 | 确保两个容器在同一Docker网络中 |
| 数据损坏 | 删除卷重新初始化：`docker volume rm airy_postgres-data` |

---

## 十一、配置问题

### Q4: 如何配置多个LLM提供商？

**场景**: 同时使用OpenAI和DeepSeek，根据任务复杂度自动选择。

**解决方案**:

编辑 `config/llm.yaml`:

```yaml
providers:
  primary:
    name: openai
    api_key: ${AIRY_LLM_API_KEY}
    base_url: https://api.openai.com/v1
    model: gpt-4-turbo
    max_tokens: 4096
    temperature: 0.7

  secondary:
    name: deepseek
    api_key: ${DEEPSEEK_API_KEY}
    base_url: https://api.deepseek.com/v1
    model: deepseek-chat
    max_tokens: 4096
    temperature: 0.7

routing_strategy:
  type: complexity_based  # 或 cost_based, round_robin
  threshold: 0.7  # 复杂度阈值，超过使用primary

cost_optimization:
  enabled: true
  budget_per_day: 50.0  # 每日预算(美元)
  prefer_secondary_for_simple_tasks: true
```

---

### Q5: 记忆系统占用过多磁盘空间怎么办？

**症状**: 磁盘空间快速增长，主要被PostgreSQL占用。

**解决方案**:

```bash
# 1. 查看记忆数据占用
docker exec agentrt-postgres-dev psql -U agentrt -d agentrt -c "
  SELECT schemaname, relname, pg_total_relation_size(relid) as size
  FROM pg_stat_user_tables
  ORDER BY size DESC
  LIMIT 10;
"

# 2. 配置自动清理策略
# 编辑 config/memory.yaml
memoryrovol:
  l1:
    ttl_seconds: 3600  # L1记忆1小时过期
    max_entries: 10000
  l2:
    ttl_seconds: 86400  # L2记忆1天过期
    max_entries: 100000
  auto_compaction:
    enabled: true
    schedule: "0 3 * * *"  # 每天凌晨3点执行
    retention_days: 30  # 保留30天数据

# 3. 手动清理过期数据
docker exec agentrt-kernel-dev python3 -m agentrt.tools.memory_cleanup --dry-run
# 确认无误后执行：
docker exec agentrt-kernel-dev python3 -m agentrt.tools.memory_cleanup
```

---

### Q6: 如何调整安全级别？

**场景**: 开发环境需要更宽松的安全策略以提高效率。

**解决方案**:

编辑 `docker/.env`:

```env
# 开发环境（宽松模式）
AIRY_PERMISSION_MODE=relaxed
AIRY_SANITIZER_MODE=relaxed
AIRY_AUDIT_ENABLED=false

# 生产环境（严格模式，推荐）
AIRY_PERMISSION_MODE=strict
AIRY_SANITIZER_MODE=strict
AIRY_AUDIT_ENABLED=true
```

**各模式对比**:

| 特性 | strict | relaxed | disabled |
|------|--------|---------|----------|
| 权限检查 | 全部检查 | 仅关键操作 | 不检查 |
| 输入净化 | 全部净化 | 仅危险字符 | 不净化 |
| 审计日志 | 全部记录 | 仅异常事件 | 不记录 |
| 性能影响 | ~5% | ~1% | 0% |
| **安全性** | ✅ 生产级 | ⚠️ 开发用 | ❌ 危险 |

⚠️ **警告**: 生产环境严禁使用 `disabled` 模式！

---

## 十二、运行时问题

### Q7: 任务卡在"running"状态不结束？

**症状**: 任务长时间显示running状态，无进度更新。

**排查步骤**:

```bash
# 1. 查看任务详情
curl http://localhost:8080/api/v1/tasks/<task_id> | jq .

# 2. 查看内核实时日志
docker logs -f agentrt-kernel-dev 2>&1 | grep -E "(task_id=<task_id>|ERROR|WARN)"

# 3. 检查LLM服务状态
curl http://localhost:8001/health

# 4. 检查系统资源使用
docker stats agentrt-kernel-dev --no-stream

# 5. 取消卡住的任务
curl -X POST http://localhost:8080/api/v1/tasks/<task_id>/cancel
```

**常见原因**:

| 原因 | 现象 | 解决方案 |
|------|------|---------|
| LLM API超时 | 任务停在cognition/planning阶段 | 增加 `AIRY_LLM_TIMEOUT` |
| 工具执行阻塞 | 任务停在action阶段 | 检查工具服务是否正常 |
| 死锁 | CPU使用率100%，无日志输出 | 重启内核服务 |
| 内存不足 | OOM Killer终止进程 | 增加内存限制 |

---

### Q9: OpenLab无法连接到内核API？

**症状**: OpenLab页面显示"无法连接到后端服务"。

**排查步骤**:

```bash
# 1. 检查OpenLab容器是否能解析kernel主机名
docker exec agentrt-openlab-dev ping kernel -c 2

# 2. 检查OpenLab的环境变量
docker exec agentrt-openlab-dev env | grep AIRY_IPC_URL

# 3. 手动从OpenLab容器测试连接
docker exec agentrt-openlab-dev curl http://kernel:8080/health

# 4. 检查内核容器的端口映射
docker port agentrt-kernel-dev

# 5. 如果使用host网络模式，确保防火墙允许
sudo ufw allow 8080/tcp
```

**常见配置错误**:

```yaml
# ❌ 错误：使用了localhost（在容器内指向自身）
AIRY_IPC_URL=http://localhost:8080

# ✅ 正确：使用服务名（Docker DNS解析）
AIRY_IPC_URL=http://kernel:8080
```

---

## 十三、安全问题

### Q10: 如何更换数据库密码？

**步骤**:

```bash
# 1. 停止所有服务
docker compose -f docker/docker-compose.yml down

# 2. 修改.env文件
nano docker/.env
# 修改 POSTGRES_PASSWORD=新密码

# 3. （可选）备份数据
docker run --rm -v airy_postgres-data:/data alpine tar czf /backup/pg-backup.tar.gz -C /data .

# 4. 启动服务（新密码生效）
docker compose -f docker/docker-compose.yml up -d

# 5. 验证连接
docker exec agentrt-postgres-dev psql -U agentrt -d agentrt -c "SELECT 1;"
```

---

## 十四、性能问题

### Q12: 系统响应缓慢，如何诊断？

**诊断流程**:

```bash
# 1. 检查系统资源
docker stats --no-stream

# 2. 查看慢查询日志
docker logs agentrt-postgres-dev 2>&1 | grep "duration:"

# 3. 查看LLM API延迟
curl http://localhost:9090/metrics | grep llm_request_duration

# 4. 分析Prometheus慢请求
# 打开 Grafana → Dashboard → Airymax Performance
# 查看 P95/P99 延迟趋势图

# 5. 生成性能报告
docker exec agentrt-kernel-dev python3 -m agentrt.tools.profile --output profile-report.html
```

**常见瓶颈及优化**:

| 瓶颈 | 症状 | 优化方案 |
|------|------|---------|
| LLM API延迟高 | cognition阶段慢 | 使用缓存、切换更快模型 |
| 数据库慢查询 | memory检索慢 | 添加索引、优化查询、读写分离 |
| 内存压力高 | GC频繁、OOM | 增加内存、优化数据结构 |
| CPU密集计算 | action阶段慢 | 并行化、使用更高效算法 |
| 网络I/O瓶颈 | IPC延迟高 | 批量发送、压缩消息 |

---

### Q13: 如何进行容量规划？

**计算公式**:

```python
# QPS估算
expected_qps = concurrent_users * requests_per_user

# 内存需求（每个任务实例）
memory_per_instance = base_memory + (avg_task_memory * max_concurrent_tasks)

# 数据库存储（每年）
storage_per_year = (
    daily_tasks * avg_task_size_bytes * 365
    + memory_l2_l3_growth_rate * 365
) * replication_factor

# 示例：1000并发用户，每用户每天10个任务
expected_qps = 1000 * (10 / 86400) ≈ 0.12 QPS（平均）
peak_qps = expected_qps * 10 ≈ 1.2 QPS（峰值，10倍突发）

memory_needed = 512MB + (50MB * 100) = 5.5GB（支持100并发任务）
storage_yearly = (10000 * 100KB * 365 + 1GB * 365) * 3 ≈ 4TB（3副本）
```

**生产环境推荐配置**:

| 规模 | CPU | 内存 | 存储 | 实例数 |
|------|-----|------|------|-------|
| 小型 (<100用户) | 4核 | 8GB | 100GB SSD | 1 |
| 中型 (100-1000用户) | 8核 | 32GB | 1TB SSD | 3 |
| 大型 (>1000用户) | 16核+ | 64GB+ | 10TB+ SSD | 5+ |

---

## 十五、升级与迁移

### Q14: 如何从v0.x升级到v1.0？

**升级前准备**:

```bash
# 1. 完整备份
./scripts/backup.sh --full --output ./backups/pre-upgrade-$(date +%Y%m%d)

# 2. 记录当前版本
curl http://localhost:8080/health | jq '.version'

# 3. 查看变更日志
cat ../../CHANGELOG.md | grep -A50 "## [1.0.0]"
```

**升级步骤**:

```bash
# 1. 拉取新代码
git pull origin main
git submodule update --init --recursive

# 2. 更新镜像
docker compose -f docker/docker-compose.yml pull
docker compose -f docker/docker-compose.yml build

# 3. 执行数据库迁移
docker compose -f docker/docker-compose.yml exec postgres psql -U agentrt -d agentrt -f /migrations/v1.0.0.sql

# 4. 滚动更新（零停机）
docker compose -f docker/docker-compose.yml up -d --no-deps --build kernel

# 5. 验证升级
curl http://localhost:8080/health
./scripts/smoke-test.sh
```

**回滚方案**（如果升级失败）:

```bash
# 1. 停止新版本
docker compose -f docker/docker-compose.yml down

# 2. 恢复备份
./scripts/restore.sh --input ./backups/pre-upgrade-YYYYMMDD

# 3. 启动旧版本
git checkout v0.x.y
docker compose -f docker/docker-compose.yml up -d
```

---

## 十六、获取帮助

如果以上文档未能解决您的问题：

### 社区支持

- **GitCode Issues**: https://gitcode.com/spharx/agentrt/issues
- **讨论区**: https://gitcode.com/spharx/agentrt/discussions

### 商业支持

- **技术支持邮箱**: support@spharx.cn
- **紧急热线**: （请联系您的客户经理）

### 报告Bug时的必填信息

为了快速定位和解决问题，请提供：

1. **Airymax版本**: `agentrt-kernel --version`
2. **操作系统**: `uname -a`
3. **Docker版本**: `docker --version`
4. **复现步骤**: 最小化复现代码
5. **期望行为 vs 实际行为**
6. **相关日志**:
   ```bash
   docker logs agentrt-kernel-dev > kernel.log 2>&1
   docker logs agentrt-postgres-dev > postgres.log 2>&1
   ```
7. **环境变量**（脱敏后）:
   ```bash
   env | grep AGENTRT > env.txt  # 删除敏感信息后附上
   ```

---

## 参考文档

| 文档 | 路径 |
|------|------|
| 快速入门 | [quickstart_protocol_gateway.md](quickstart_protocol_gateway.md) |
| 最佳实践 | [best_practices.md](best_practices.md) |
| 协议集成 | [protocol_integration_guide.md](protocol_integration_guide.md) |
| 部署指南 | [deployment.md](deployment.md) |
| 安全加固 | [security-hardening.md](security-hardening.md) |
