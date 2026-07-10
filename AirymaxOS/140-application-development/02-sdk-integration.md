Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# SDK 集成设计

> **文档定位**：agentrt-linux（AirymaxOS，极境智能体操作系统）Agent 应用开发体系核心子文档，定义 Python/Rust/Go/TypeScript 四语言 SDK 与 agentrt-linux 内核通信的集成设计\
> **版本**：0.1.1（文档体系完成）/ 1.0.1（开发）\
> **最后更新**：2026-07-09\
> **理论根基**：Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论\
> **SPDX-License-Identifier**：AGPL-3.0-or-later OR Apache-2.0\
> **同源映射**：agentrt 四语言 SDK（IRON-9 v2 [SC] 共享契约层共享头文件 + [SS] 语义同源层 SDK 层签名同源）\
> **IRON-9 v2 层次**：[SC] 共享契约层（IPC 消息头结构、syscall 编号、错误码完全共享）+ [SS] 语义同源层（SDK 层签名同源，同一份源码两端编译，构建期条件编译切换传输层；其他层语义同源）

---

## 1. 设计目标与架构

### 1.1 设计目标

agentrt-linux（AirymaxOS）提供四语言 SDK（Python / Rust / Go / TypeScript），统一封装与内核的通信细节，使上层 Agent 应用无需关心底层 syscall 或 AgentsIPC 协议。SDK 集成设计达成以下工程目标：

1. **零拷贝高性能**：通过 io_uring 提交队列与内核共享内存，避免数据拷贝
2. **FFI 边界安全**：四语言与 C ABI 边界严格类型检查，杜绝内存安全漏洞
3. **同源 API 一致性**：四语言 SDK（SDK 层）API 签名完全一致，遵循 IRON-9 v2 [SS] 层
4. **运行时环境感知**：SDK 自动检测宿主为 agentrt-linux 或 agentrt，切换通信路径
5. **统一错误码**：四语言共享 `include/airymax/error.h` 错误码（[SC] 层）

### 1.2 SDK 分层架构

```
┌─────────────────────────────────────────────────────────┐
│ L5: Agent 应用（业务逻辑）                              │
├─────────────────────────────────────────────────────────┤
│ L4: 嵌套客户端（CognitionClient/SafetyClient/...16 个） │
├─────────────────────────────────────────────────────────┤
│ L3: SDK 核心 API（四语言同源）                          │
├──────────┬──────────┬──────────┬───────────────────────┤
│ Python   │ Rust     │ Go       │ TypeScript            │
│ CFFI     │ bindgen  │ cgo      │ N-API                 │
├──────────┴──────────┴──────────┴───────────────────────┤
│ L2: libagentrt.so（C 共享库，FFI 边界）                 │
├─────────────────────────────────────────────────────────┤
│ L1: syscall / AgentsIPC（io_uring）                      │
├─────────────────────────────────────────────────────────┤
│ L0: agentrt-linux 内核                                   │
└─────────────────────────────────────────────────────────┘
```

### 1.3 IRON-9 v2 共享层次

| 层次 | 共享内容 | agentrt-linux SDK | agentrt SDK |
|------|----------|-------------------|-------------|
| [SC] | IPC 消息头结构、syscall 编号、错误码、规则编号体系 | 完全共享 | 完全共享 |
| [SS] | CognitionClient/SafetyClient 等 16 客户端 API 签名 | SDK 层签名同源，实现独立 | SDK 层签名同源，实现独立 |
| [IND] | FFI 绑定层、语言特定包装 | 各自独立 | 各自独立 |

---

## 2. AgentsIPC 128B 消息头

### 2.1 消息头结构

AgentsIPC 协议的核心是 128 字节定长消息头，定义于 IRON-9 v2 [SC] 共享契约层：

```c
/* include/uapi/agentrt/ipc.h（IRON-9 v2 [SC] 共享契约层）
 * 与 agentrt 用户态运行时完全共享，禁止单端修改 */
struct agentrt_ipc_msg_hdr_t {
	uint32_t magic;          /* 0x41524531 ('ARE1') */
	uint16_t version;        /* 协议版本，当前 1 */
	uint16_t type;           /* REQUEST/RESPONSE/EVENT/STREAM/CONTROL */
	uint32_t src_agent_id;   /* 发送方 Agent ID */
	uint32_t dst_agent_id;   /* 接收方 Agent ID */
	uint64_t payload_len;    /* payload 长度（可超出 128B） */
	uint64_t trace_id;       /* 链路追踪 ID（贯穿全链路） */
	uint32_t seq;            /* 序列号（请求-响应配对） */
	uint32_t flags;          /* 标志位（压缩/加密/重试） */
	uint64_t reserved[8];    /* 预留 64 字节扩展 */
};  /* 共 128 字节，编译期 static_assert 验证 */
static_assert(sizeof(struct agentrt_ipc_msg_hdr_t) == 128,
	      "IPC header must be exactly 128 bytes");
```

### 2.2 消息类型

```c
enum agentrt_ipc_msg_type {
	AGENTRT_IPC_TYPE_REQUEST  = 1,  /* 请求 */
	AGENTRT_IPC_TYPE_RESPONSE = 2,  /* 响应 */
	AGENTRT_IPC_TYPE_EVENT    = 3,  /* 事件（单向） */
	AGENTRT_IPC_TYPE_STREAM   = 4,  /* 流式（LLM 流式输出） */
	AGENTRT_IPC_TYPE_CONTROL  = 5,  /* 控制消息（心跳/关闭） */
};
```

### 2.3 五种 payload 类型

| Payload 类型 | 用途 | 编码 |
|--------------|------|------|
| JSON | 结构化数据（请求参数） | UTF-8 JSON |
| Protobuf | 高性能二进制 | Proto3 |
| Raw Bytes | 原始字节（向量） | 直接字节 |
| Stream Chunk | 流式分片（LLM Token） | 分片头 + 数据 |
| Control | 控制信号 | 固定结构 |

---

## 3. FFI 边界设计

### 3.1 C 共享库接口

`libagentrt.so` 是四语言 SDK 的统一 C 边界，所有跨语言调用均经此：

```c
/* include/agentrt/sdk.h（[SS] 语义同源层） */
typedef struct agentrt_client agentrt_client_t;
typedef struct agentrt_response agentrt_response_t;

/* 客户端创建/销毁 */
agentrt_client_t *agentrt_client_create(const char *socket_path);
void agentrt_client_destroy(agentrt_client_t *client);

/* 通用请求-响应（封装 AgentsIPC） */
int agentrt_client_call(agentrt_client_t *client,
			uint32_t dst_agent_id,
			const char *method,
			const uint8_t *payload, size_t payload_len,
			agentrt_response_t **out_resp);

/* 流式请求（封装 STREAM 消息） */
int agentrt_client_call_stream(agentrt_client_t *client,
			       uint32_t dst_agent_id,
			       const char *method,
			       const uint8_t *payload, size_t payload_len,
			       void (*on_chunk)(const uint8_t *, size_t, void *),
			       void *user_data);

/* 响应释放 */
void agentrt_response_free(agentrt_response_t *resp);
```

### 3.2 内存安全约定

FFI 边界遵循「谁分配谁释放」原则，杜绝跨语言内存泄漏：

| 分配方 | 释放方 | 内存类型 |
|--------|--------|----------|
| C 库（malloc） | C 库（agentrt_response_free） | 响应数据 |
| 语言运行时 | 语言运行时 | 请求 payload |
| io_uring 共享内存 | 内核回收 | SQE/CQE |

### 3.3 错误码传递

C 负整数体系错误码直接穿透 FFI 边界：

```c
/* SDK 调用返回负错误码，语言层包装为异常 */
int ret = agentrt_client_call(client, dst, method, ...);
if (ret < 0) {
	/* ret 为 AGENTRT_E* 负错误码，[SC] 层共享 */
	return ret;
}
```

---

## 4. 四语言 SDK 客户端实现

### 4.1 Python SDK（CFFI）

```python
# agentrt_python/_core.py
import cffi

ffi = cffi.FFI()
ffi.cdef("""
    typedef struct agentrt_client agentrt_client_t;
    typedef struct agentrt_response agentrt_response_t;
    agentrt_client_t *agentrt_client_create(const char *socket_path);
    void agentrt_client_destroy(agentrt_client_t *client);
    int agentrt_client_call(agentrt_client_t *client,
                            uint32_t dst_agent_id,
                            const char *method,
                            const uint8_t *payload, size_t payload_len,
                            agentrt_response_t **out_resp);
    const uint8_t *agentrt_response_data(agentrt_response_t *resp,
                                          size_t *out_len);
    void agentrt_response_free(agentrt_response_t *resp);
""")

lib = ffi.dlopen("libagentrt.so")

class AgentrtError(Exception):
    """统一错误码（[SC] 层共享）"""
    def __init__(self, code: int):
        self.code = code
        super().__init__(f"agentrt error: {code}")

class Client:
    """SDK 客户端，封装 syscall/AgentsIPC 通信"""
    def __init__(self, socket_path: str = "/var/run/agentrt/agent.sock"):
        self._client = lib.agentrt_client_create(
            ffi.new("char[]", socket_path.encode()))
        if not self._client:
            raise AgentrtError(-1)

    def call(self, dst_agent_id: int, method: str, payload: bytes) -> bytes:
        buf = ffi.from_buffer(payload) if payload else ffi.NULL
        resp_ptr = ffi.new("agentrt_response_t **")
        ret = lib.agentrt_client_call(
            self._client, dst_agent_id,
            method.encode(), buf, len(payload), resp_ptr)
        if ret < 0:
            raise AgentrtError(ret)
        try:
            length_ptr = ffi.new("size_t *")
            data = lib.agentrt_response_data(resp_ptr[0], length_ptr)
            return ffi.buffer(data, length_ptr[0])[:]
        finally:
            lib.agentrt_response_free(resp_ptr[0])

    def __del__(self):
        if self._client:
            lib.agentrt_client_destroy(self._client)
```

### 4.2 Rust SDK（bindgen）

```rust
// agentrt_rust/src/lib.rs
use std::ffi::{CStr, CString};
use std::os::raw::{c_char, c_uint};
use std::ptr;

/* bindgen 自动生成（[IND] 完全独立层） */
mod ffi {
    #![allow(non_upper_case_globals, non_camel_case_types)]
    include!(concat!(env!("OUT_DIR"), "/bindings.rs"));
}

#[derive(Debug)]
pub enum AgentrtError {
    InvalidArg,
    NoMemory,
    NotFound,
    BudgetExhausted,
    Unknown(i32),
}

impl From<i32> for AgentrtError {
    fn from(code: i32) -> Self {
        /* [SC] 层错误码映射 */
        match code {
            -2 => AgentrtError::InvalidArg,
            -12 => AgentrtError::NoMemory,
            -101 => AgentrtError::NotFound,
            -401 => AgentrtError::BudgetExhausted,
            _ => AgentrtError::Unknown(code),
        }
    }
}

pub struct Client {
    inner: *mut ffi::agentrt_client_t,
}

impl Client {
    pub fn new(socket_path: &str) -> Result<Self, AgentrtError> {
        let c_path = CString::new(socket_path).unwrap();
        unsafe {
            let inner = ffi::agentrt_client_create(c_path.as_ptr());
            if inner.is_null() {
                return Err(AgentrtError::Unknown(-1));
            }
            Ok(Client { inner })
        }
    }

    pub fn call(&self, dst_agent_id: u32, method: &str,
                payload: &[u8]) -> Result<Vec<u8>, AgentrtError> {
        let c_method = CString::new(method).unwrap();
        let mut resp_ptr: *mut ffi::agentrt_response_t = ptr::null_mut();
        let ret = unsafe {
            ffi::agentrt_client_call(
                self.inner, dst_agent_id, c_method.as_ptr(),
                payload.as_ptr(), payload.len(), &mut resp_ptr)
        };
        if ret < 0 {
            return Err(AgentrtError::from(ret));
        }
        unsafe {
            let mut len: usize = 0;
            let data = ffi::agentrt_response_data(resp_ptr, &mut len);
            let result = std::slice::from_raw_parts(data, len).to_vec();
            ffi::agentrt_response_free(resp_ptr);
            Ok(result)
        }
    }
}

impl Drop for Client {
    fn drop(&mut self) {
        unsafe { ffi::agentrt_client_destroy(self.inner); }
    }
}
```

### 4.3 Go SDK（cgo）

```go
// agentrt_go/client.go
package agentrt

/*
#cgo CFLAGS: -I${SRCDIR}/include
#cgo LDFLAGS: -L${SRCDIR}/lib -lagentrt
#include <agentrt/sdk.h>
*/
import "C"
import (
	"errors"
	"unsafe"
)

type Client struct {
	handle *C.agentrt_client_t
}

type AgentrtError struct{ Code int32 }

func (e *AgentrtError) Error() string {
	return errors.New("agentrt error").Error()
}

func NewClient(socketPath string) (*Client, error) {
	cPath := C.CString(socketPath)
	defer C.free(unsafe.Pointer(cPath))
	h := C.agentrt_client_create(cPath)
	if h == nil {
		return nil, &AgentrtError{Code: -1}
	}
	return &Client{handle: h}, nil
}

func (c *Client) Call(dstAgentID uint32, method string,
	payload []byte) ([]byte, error) {
	cMethod := C.CString(method)
	defer C.free(unsafe.Pointer(cMethod))

	var resp *C.agentrt_response_t
	var payloadPtr *C.uint8_t
	if len(payload) > 0 {
		payloadPtr = (*C.uint8_t)(unsafe.Pointer(&payload[0]))
	}
	ret := C.agentrt_client_call(c.handle, C.uint(dstAgentID),
		cMethod, payloadPtr, C.size_t(len(payload)), &resp)
	if ret < 0 {
		return nil, &AgentrtError{Code: int32(ret)}
	}
	defer C.agentrt_response_free(resp)

	var length C.size_t
	data := C.agentrt_response_data(resp, &length)
	return C.GoBytes(unsafe.Pointer(data), C.int(length)), nil
}

func (c *Client) Close() {
	if c.handle != nil {
		C.agentrt_client_destroy(c.handle)
	}
}
```

### 4.4 TypeScript SDK（N-API）

```typescript
// agentrt_typescript/native.ts
import { addon, AgentrtError } from './binding';

/**
 * SDK 客户端（封装 libagentrt.so 调用）
 * API 签名与 Python/Rust/Go 完全一致（[SS] 语义同源层，SDK 层签名同源）
 */
export class Client {
	private handle: Buffer;

	constructor(socketPath: string = '/var/run/agentrt/agent.sock') {
		this.handle = addon.clientCreate(socketPath);
		if (!this.handle) {
			throw new AgentrtError(-1);
		}
	}

	call(dstAgentId: number, method: string, payload: Buffer): Buffer {
		const resp = addon.clientCall(this.handle, dstAgentId,
			method, payload);
		if (resp.error < 0) {
			throw new AgentrtError(resp.error);
		}
		return resp.data;
	}

	async callStream(dstAgentId: number, method: string,
		payload: Buffer): Promise<Buffer[]> {
		const chunks: Buffer[] = [];
		await addon.clientCallStream(this.handle, dstAgentId,
			method, payload,
			(chunk: Buffer) => chunks.push(chunk));
		return chunks;
	}

	close(): void {
		if (this.handle) {
			addon.clientDestroy(this.handle);
		}
	}
}
```

---

## 5. syscall 调用示例

### 5.1 SDK 内部 syscall 路径

当 SDK 检测到宿主为 agentrt-linux 时，自动切换至 syscall 路径：

```c
/* libagentrt/sdk.c 内部实现 */
int agentrt_client_call(agentrt_client_t *client,
			uint32_t dst_agent_id,
			const char *method,
			const uint8_t *payload, size_t payload_len,
			agentrt_response_t **out_resp)
{
	struct agentrt_ipc_msg_hdr_t hdr = {0};
	int ret;

	if (!client || !method || (!payload && payload_len > 0))
		return -AGENTRT_EINVAL;

	/* 填充 128B 消息头 */
	hdr.magic = 0x41524531;
	hdr.version = 1;
	hdr.type = AGENTRT_IPC_TYPE_REQUEST;
	hdr.src_agent_id = client->self_agent_id;
	hdr.dst_agent_id = dst_agent_id;
	hdr.payload_len = payload_len;
	hdr.trace_id = client->next_trace_id++;
	hdr.seq = client->next_seq++;

	/* agentrt-linux 路径：syscall（io_uring 注册） */
	if (client->use_syscall) {
		ret = syscall(AGENTRT_SYS_IPC_SEND, &hdr, payload, payload_len);
		if (ret < 0)
			return ret;
		return agentrt_syscall_recv(client, hdr.seq, out_resp);
	}

	/* agentrt 用户态路径：io_uring 提交至消息队列 */
	return agentrt_ipc_uring_submit(client, &hdr, payload, payload_len,
					out_resp);
}
```

### 5.2 SDK 客户端调用系统调用示例

以下展示一个 Python SDK 客户端完整调用 `AGENTRT_SYS_COGNITION_PROCESS` 系统调用的代码片段：

```python
# agentrt_python/clients/cognition.py
from ._core import Client, AgentrtError
import json

class CognitionClient:
    """认知客户端（16 嵌套客户端之一）"""
    LLM_DAEMON_ID = 2  # llm_d 服务 Agent ID

    def __init__(self, token_budget: int = 10000, client: Client = None):
        self._client = client or Client()
        self._token_budget = token_budget
        self._tokens_used = 0

    def process(self, prompt: str, **kwargs) -> dict:
        """
        执行认知处理（封装 syscall AGENTRT_SYS_COGNITION_PROCESS）
        当宿主为 agentrt-linux 时，SDK 内部走 syscall 路径
        """
        if self._tokens_used >= self._token_budget:
            raise AgentrtError(-401)  # AGENTRT_EBUDGET_EXHAUSTED

        payload = json.dumps({
            "prompt": prompt,
            "token_budget": self._token_budget - self._tokens_used,
            "options": kwargs,
        }).encode()

        # SDK 内部检测宿主：
        #   - agentrt-linux -> syscall(AGENTRT_SYS_COGNITION_PROCESS)
        #   - agentrt 用户态 -> AgentsIPC 调用 llm_d
        resp = self._client.call(self.LLM_DAEMON_ID,
                                  "cognition.process", payload)
        result = json.loads(resp)

        self._tokens_used += result.get("tokens_used", 0)
        return result

    def process_stream(self, prompt: str, **kwargs):
        """流式认知处理（封装 STREAM 类型 IPC 消息）"""
        payload = json.dumps({"prompt": prompt, "options": kwargs}).encode()
        for chunk in self._client.call_stream(self.LLM_DAEMON_ID,
                                               "cognition.stream", payload):
            yield json.loads(chunk)
```

### 5.3 完整调用时序

```
[Python 应用]
   CognitionClient.process("hello")
       |
       v
[libagentrt.so]
   agentrt_client_call()
       |
       v (检测宿主为 agentrt-linux)
   syscall(AGENTRT_SYS_IPC_SEND, hdr, payload)
       |
       v
[内核态 agentrt_core]
   io_uring 提交队列接收 SQE
       |
       v
[内核态 SCHED_AGENT]
   调度 llm_d kthread 处理
       |
       v
[llm_d daemon]
   解析 128B 消息头 + payload
   执行 LLM 推理
       |
       v (响应回程)
[内核态]
   io_uring 完成队列 CQE
       |
       v
[libagentrt.so]
   agentrt_syscall_recv() 取回响应
       |
       v
[Python 应用]
   返回 dict 结果
```

---

## 6. 16 嵌套客户端清单

四语言 SDK 各提供 4 大客户端，共 16 嵌套客户端（SDK 层 API 签名 [SS] 同源）：

| 客户端 | 子客户端 | 主要方法 | 目标 daemon |
|--------|----------|----------|-------------|
| CognitionClient | CognitionStreamClient | process / process_stream | llm_d |
| | CognitionReflectClient | reflect / critique | llm_d |
| | CognitionPlanClient | plan / decompose | llm_d |
| | CognitionCoordClient | coordinate / dispatch | sched_d |
| SafetyClient | PermissionClient | check / grant / revoke | Cupolas |
| | SandboxClient | enter / exit / exec | tool_d |
| | AuditClient | log / query / verify | monit_d |
| | CryptoClient | sign / verify / encrypt | Cupolas |
| ToolClient | ToolRegistryClient | register / list / unregister | market_d |
| | ToolExecClient | execute / validate / compensate | tool_d |
| | ToolMarketClient | search / install / publish | market_d |
| | ToolVersionClient | pin / upgrade / rollback | market_d |
| ChatClient | ChatSessionClient | open / send / close | gateway_d |
| | ChatStreamClient | stream / interrupt / resume | gateway_d |
| | ChatHistoryClient | save / load / search | vfs_d |
| | ChatProtocolClient | mcp / a2a / openai_compat | gateway_d |

---

## 7. 运行时环境感知

### 7.1 宿主检测机制

SDK 在初始化时检测宿主环境，选择通信路径：

```c
/* libagentrt/host_detect.c */
bool agentrt_host_is_airymaxos(void)
{
	/* 检测 /proc/airymaxos/version 是否存在 */
	int fd = open("/proc/airymaxos/version", O_RDONLY);
	if (fd < 0)
		return false;
	close(fd);
	return true;
}
```

### 7.2 路径选择决策表

| 宿主 | 通信路径 | 传输机制 | 性能 |
|------|----------|----------|------|
| agentrt-linux | syscall | io_uring 零拷贝 | 最高 |
| agentrt 用户态 | AgentsIPC | 用户态消息队列 | 高 |
| 主流 Linux 发行版 | Unix socket | 回环 socket | 中 |

---

## 8. 错误处理与重试

### 8.1 错误码映射

四语言 SDK 共享 `include/airymax/error.h`（[SC] 层），各语言包装为本地异常类型：

| C 错误码 | Python 异常 | Rust 变体 | Go 错误 | TS 异常 |
|----------|-------------|-----------|---------|---------|
| -2 EINVAL | ValueError | InvalidArg | ErrInvalid | TypeError |
| -12 ENOMEM | MemoryError | NoMemory | ErrOOM | RangeError |
| -101 ENOENT | KeyError | NotFound | ErrNotFound | NotFoundError |
| -401 EBUDGET | BudgetError | BudgetExhausted | ErrBudget | BudgetError |
| -701 EPERM | PermissionError | Permission | ErrPerm | PermissionError |

### 8.2 自动重试策略

```python
# SDK 内置重试（指数退避）
def call_with_retry(client, dst, method, payload, max_retries=3):
    for attempt in range(max_retries):
        try:
            return client.call(dst, method, payload)
        except AgentrtError as e:
            if e.code in (-701, -401):  # 权限/预算错误不重试
                raise
            if attempt == max_retries - 1:
                raise
            time.sleep(0.1 * (2 ** attempt))  # 指数退避
```

---

## 9. 测试与验证

### 9.1 跨语言一致性测试

四语言 SDK 必须通过同一套测试用例验证 API 行为一致：

| 测试维度 | 验证方法 | 通过标准 |
|----------|----------|----------|
| SDK API 签名 | golden 文件对比 | 四语言签名完全一致 |
| 错误码映射 | 错误码矩阵测试 | 全部错误码正确穿透 |
| 128B 消息头 | 字节级对比 | magic/version/trace_id 一致 |
| 流式分片 | 顺序完整性 | 分片顺序与顺序一致 |

### 9.2 FFI 边界安全测试

```bash
# Rust SDK 用 Miri 检测未定义行为
cargo miri test

# Python SDK 用 valgrind 检测内存泄漏
valgrind --leak-check=full python -m pytest tests/

# Go SDK 用 race detector
go test -race ./...
```

---

## 10. 相关文档

- `140-application-development/README.md`（应用开发主索引）
- `140-application-development/01-agent-lifecycle.md`（Agent 生命周期管理）
- `30-interfaces/02-ipc-protocol.md`（AgentsIPC 协议详细规范）
- `30-interfaces/03-sdk-api.md`（SDK API 接口设计）
- `160-compatibility/04-sdk-compatibility.md`（SDK 跨版本兼容性）

---

## 11. 参考材料

- Linux 6.6 `io_uring` 子系统（异步 I/O 共享内存）
- Python CFFI 项目（C Foreign Function Interface）
- Rust bindgen 项目（C 头文件绑定生成）
- Go cgo 文档（C 互操作）
- Node.js N-API 文档（原生插件接口）
- agentrt 四语言 SDK 实现（IRON-9 v2 [SC]/[SS] 同源）

---

> **文档结束** | agentrt-linux（AirymaxOS）SDK 集成设计 v0.1.1
> 遵循 IRON-9 v2 [SC] 共享契约层 + [SS] 语义同源层与 agentrt SDK 同源
