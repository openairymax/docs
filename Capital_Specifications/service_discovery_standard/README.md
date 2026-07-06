# Airymax 服务发现规范

> **版本**: v0.1.0-draft | **状态**: 草案 | **日期**: 2026-07-04
> **版权归属人**: SPHARX Ltd. | **许可证**: AGPL-3.0-or-later OR Apache-2.0（双许可证，SPHARX Ltd.）

---

## 1. 现状问题（已验证）

### 1.1 完全基于共享内存（CONFIRMED）
- `daemon/common/include/service_discovery.h:6-18` 首段注释："基于共享内存的跨进程服务注册中心"
- 设计原则第 1 条："零依赖：不依赖外部注册中心（如 etcd/consul）"
- `#define SD_SHM_NAME "/agentos_service_registry"`
- 无 DNS-SD/Consul/etcd 等标准协议适配器

### 1.2 问题影响
- 单机部署性能优秀，但无法跨节点发现
- 与云原生生态（K8s/Consul/etcd）不兼容
- 第三方实现无法复用标准服务发现

## 2. 标准规范

### 2.1 多后端适配器架构（ARE-SD-01）

```
service_discovery (统一接口)
├── shm_adapter        // 共享内存（高性能本地，保留）
├── dns_sd_adapter     // DNS-SD/mDNS（零配置局域网）
├── consul_adapter     // HashiCorp Consul（云原生）
├── etcd_adapter       // CNCF etcd（K8s 原生）
└── k8s_adapter        // Kubernetes API（原生服务）
```

### 2.2 统一接口（ARE-SD-02）

```c
typedef struct are_svc_discovery_adapter {
    const char *name;  // "shm"/"dns-sd"/"consul"/"etcd"/"k8s"
    int (*register_service)(const char *name, const char *endpoint, uint16_t port);
    int (*discover_service)(const char *name, char *endpoint, size_t len);
    int (*unregister_service)(const char *name);
    int (*list_services)(are_svc_entry_t *entries, size_t *count);
    void *context;  // 适配器私有上下文
} are_svc_discovery_adapter_t;
```

### 2.3 配置选择

```yaml
service_discovery:
  backend: shm  # 默认（高性能本地）
  # backend: consul
  # backend: etcd
  # backend: k8s
  shm:
    name: /agentos_service_registry
    size: 1048576
  consul:
    address: http://consul:8500
  etcd:
    endpoints: [http://etcd:2379]
  k8s:
    namespace: agentrt
```

## 3. 修复任务

详见 [0.1.1详细任务清单.md](../../../DocsClosed/0.1.1详细任务清单.md) P0.23（服务发现标准协议适配器，20h）

---

<div align="center">

**© 2025-2026 SPHARX Ltd.**

</div>
