Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux 统一错误码参考

> **文档定位**： agentrt-linux（AirymaxOS）全系统统一错误码的权威参考，定义内核态与用户态双错误码体系、分段规划、与 agentrt 的映射关系、使用规范与维护流程\
> **版本**： 0.1.1（文档体系完成）/ 1.0.1（开发）\
> **最后更新**： 2026-07-07\
> **父文档**： [项目管理规范总览](README.md)\
> **关联规范**： IRON-9 v2 工程铁律（内部工程标准规范） / [五维正交 24 原则](../../10-architecture/02-five-dimensional-principles.md) / [接口设计规范](../../30-interfaces/README.md)

---

## 1. 概述

### 1.1 设计目标

agentrt-linux（AirymaxOS）统一错误码体系的设计目标：

| 目标 | 说明 | 对应原则 |
|------|------|----------|
| **全系统一致** | 内核态与用户态错误码可关联追溯，调试时获得完整错误链 | E-6 错误可追溯原则 |
| **分段无冲突** | 各子仓错误码区间独立，避免数值冲突 | K-2 接口契约化原则 |
| **与 agentrt 对齐** | [SC] 层错误码完全共享，[SS] 层语义一致 | IRON-9 v2 |
| **可扩展** | 预留扩展空间，支持未来新子仓和新错误类型 | A-1 简约至上原则 |
| **可审计** | 错误码定义、使用、变更全程可追溯 | S-1 反馈闭环原则 |

### 1.2 适用范围

本错误码体系适用于 agentrt-linux（AirymaxOS）的以下全部子仓和组件：

| 子仓 / 组件 | 错误码空间 | 说明 |
|-------------|------------|------|
| airymaxos-kernel | 内核态负整数 (-1 ~ -899) | 内核模块、系统调用返回值 |
| airymaxos-services | 用户态十六进制 (0x0XXX0000) | 用户态守护进程、服务接口返回值 |
| airymaxos-security | 用户态十六进制 (0x3XXX0000) | 安全模块返回值 |
| airymaxos-memory | 用户态十六进制 (0x4XXX0000) | 记忆子系统返回值 |
| airymaxos-cognition | 用户态十六进制 (0x5XXX0000) | 认知运行时返回值 |
| airymaxos-cloudnative | 用户态十六进制 (0x6XXX0000) | 云原生组件返回值 |
| airymaxos-system | 用户态十六进制 (0x7XXX0000) | 系统工具返回值 |
| airymaxos-tests-linux | 用户态十六进制 (0x8XXX0000) | 测试框架返回值 |

---

## 2. 双错误码体系

### 2.1 设计原理

agentrt-linux（AirymaxOS）采用双错误码体系的核心原因在于内核态与用户态的错误表示本质不同：

| 特性 | 内核态 | 用户态 |
|------|--------|--------|
| **表示方式** | 负整数（C 语言惯用法，返回 -EXXX） | 十六进制（便于位操作和分类） |
| **传输方式** | 系统调用返回值（errno） | 函数返回值、IPC 消息字段 |
| **编码空间** | 紧凑（-1 ~ -899，共 899 个有效值） | 宽松（0x00010000 ~ 0xFFFF0000，共 65535 个有效值） |
| **扩展性** | 有限（每个分段 100 个值） | 充裕（每个分段 4096 个值） |
| **与 C 标准兼容** | 负数 errno 直接兼容 POSIX | 需转换为负数 errno 或自定义枚举 |

### 2.2 内核态错误码：负整数体系

内核态错误码采用负整数表示，与 Linux 内核 errno 风格一致。所有内核态错误码定义为负整数的宏。

#### 2.2.1 通用错误码（-1 ~ -99）

通用错误码复用 Linux 内核标准 errno，agentrt-linux 不重新定义，直接使用内核已有定义：

| 错误码 | 宏名称 | 含义 | 典型场景 |
|--------|--------|------|----------|
| -1 | -EPERM | 操作不允许 | 权限不足 |
| -2 | -ENOENT | 文件或目录不存在 | 路径查找失败 |
| -3 | -ESRCH | 进程不存在 | PID 查找失败 |
| -4 | -EINTR | 系统调用被中断 | 信号中断 |
| -5 | -EIO | I/O 错误 | 磁盘读写失败 |
| -6 | -ENXIO | 设备或地址不存在 | 设备节点未找到 |
| -7 | -E2BIG | 参数列表过长 | execve 参数过多 |
| -8 | -ENOEXEC | 执行格式错误 | 可执行文件格式无效 |
| -9 | -EBADF | 文件描述符无效 | fd 已关闭或无效 |
| -10 | -ECHILD | 无子进程 | waitpid 无子进程 |
| -11 | -EAGAIN | 资源暂时不可用 | 非阻塞操作需重试 |
| -12 | -ENOMEM | 内存不足 | 分配失败 |
| -13 | -EACCES | 权限不足 | 访问拒绝 |
| -14 | -EFAULT | 地址错误 | 无效指针 |
| -15 | -ENOTBLK | 需要块设备 | 非块设备操作 |
| -16 | -EBUSY | 设备或资源忙 | 资源被占用 |
| -17 | -EEXIST | 文件已存在 | 创建已存在的文件 |
| -18 | -EXDEV | 跨设备链接 | 不同文件系统间操作 |
| -19 | -ENODEV | 无此设备 | 设备不存在 |
| -20 | -ENOTDIR | 不是目录 | 路径组件非目录 |
| -21 | -EISDIR | 是目录 | 对目录执行非法操作 |
| -22 | -EINVAL | 参数无效 | 无效的系统调用参数 |
| -23 | -ENFILE | 文件表溢出 | 系统级文件表满 |
| -24 | -EMFILE | 打开文件过多 | 进程级 fd 表满 |
| -25 | -ENOTTY | 不适当的 ioctl | 对非终端设备执行 ioctl |
| -26 | -ETXTBSY | 文本文件忙 | 正在写入的可执行文件 |
| -27 | -EFBIG | 文件过大 | 超出文件大小限制 |
| -28 | -ENOSPC | 设备无空间 | 磁盘满 |
| -29 | -ESPIPE | 非法寻址 | 对管道进行 lseek |
| -30 | -EROFS | 只读文件系统 | 对只读 fs 写操作 |
| -31 | -EMLINK | 链接过多 | 超出最大链接数 |
| -32 | -EPIPE | 管道破裂 | 向无读端的管道写入 |
| -33 | -EDOM | 数学参数超出域 | 数学函数参数无效 |
| -34 | -ERANGE | 数学结果超出范围 | 数学函数结果溢出 |
| -35 | -EDEADLK | 资源死锁 | 检测到死锁 |
| -36 | -ENAMETOOLONG | 文件名过长 | 超出路径名长度限制 |
| -37 | -ENOLCK | 无可用锁 | 记录锁不可用 |
| -38 | -ENOSYS | 系统调用未实现 | 功能未实现 |
| -39 | -ENOTEMPTY | 目录非空 | 删除非空目录 |
| -40 | -ELOOP | 符号链接循环过多 | 符号链接嵌套过深 |
| -42 | -ENOMSG | 无所需类型消息 | 消息队列为空 |
| -43 | -EIDRM | 标识符已删除 | IPC 对象已删除 |
| -45 | -ERESTART | 需重启系统调用 | 内核内部状态 |
| -50 | -ENODATA | 无可用数据 | 流中无数据 |
| -51 | -ETIME | 定时器超时 | 定时器过期 |
| -52 | -ENOSR | 流出资源 | STREAMS 资源不足 |
| -55 | -ENOTSUP | 操作不支持 | 功能不支持 |
| -60 | -EOVERFLOW | 值过大 | 超出数据类型范围 |
| -61 | -ENOTCONN | 套接字未连接 | 对未连接套接字操作 |
| -62 | -ETIMEDOUT | 连接超时 | 网络操作超时 |
| -63 | -ECONNREFUSED | 连接被拒绝 | 远程拒绝连接 |
| -64 | -EHOSTDOWN | 主机已关机 | 目标主机不可达 |
| -65 | -EHOSTUNREACH | 主机不可达 | 路由不可达 |
| -66 | -EALREADY | 操作已在进行中 | 重复操作 |
| -67 | -EINPROGRESS | 操作正在进行中 | 非阻塞操作 |
| -68 | -ESTALE | 文件句柄过期 | NFS 文件句柄过期 |
| -69 | -ECANCELED | 操作已取消 | 异步操作取消 |

#### 2.2.2 系统错误码（-100 ~ -199）

系统错误码由 airymaxos-system 子仓定义，涵盖系统级错误场景：

| 错误码 | 宏名称 | 含义 | 典型场景 |
|--------|--------|------|----------|
| -100 | -E_SYS_PKG_NOT_FOUND | 软件包未找到 | dnf 未找到指定包 |
| -101 | -E_SYS_PKG_CONFLICT | 软件包冲突 | 依赖冲突 |
| -102 | -E_SYS_PKG_INSTALL_FAIL | 软件包安装失败 | RPM 安装错误 |
| -103 | -E_SYS_CONFIG_INVALID | 配置无效 | 配置文件语法错误 |
| -104 | -E_SYS_CONFIG_APPLY_FAIL | 配置应用失败 | systemd 配置重新加载失败 |
| -105 | -E_SYS_SERVICE_FAILED | 系统服务失败 | 服务启动或运行失败 |
| -106 | -E_SYS_SERVICE_NOT_FOUND | 系统服务未找到 | systemd 未找到指定单元 |
| -107 | -E_SYS_DEPENDENCY_MISSING | 依赖缺失 | 缺少必要依赖项 |
| -108 | -E_SYS_VERSION_MISMATCH | 版本不匹配 | 库或内核版本不兼容 |
| -109 | -E_SYS_RESOURCE_EXHAUSTED | 系统资源耗尽 | 系统级资源（如 PID）耗尽 |
| -110 | -E_SYS_BOOT_FAILED | 引导失败 | 系统启动失败 |
| -111 | -E_SYS_UPDATE_FAILED | 系统更新失败 | 系统更新过程错误 |
| -112 | -E_SYS_ROLLBACK_FAILED | 回滚失败 | 系统回滚过程错误 |
| -113 | -E_SYS_BUILD_FAILED | 构建失败 | 包构建过程错误 |
| -114 | -E_SYS_SIGNATURE_INVALID | 签名无效 | RPM 签名验证失败 |
| -115 | -E_SYS_REPO_UNAVAILABLE | 软件源不可用 | 软件仓库不可达 |
| -116 | -E_SYS_SHELL_EXEC_FAIL | Shell 执行失败 | Shell 命令执行错误 |
| -117 | -E_SYS_ENV_INVALID | 环境变量无效 | 环境变量设置错误 |
| -118 | -E_SYS_DEVSTATION_FAILED | DevStation 失败 | 开发环境初始化失败 |
| -119 | -E_SYS_LICENSE_INVALID | 许可证无效 | 软件许可证不兼容 |

#### 2.2.3 调度错误码（-200 ~ -299）

调度错误码由 airymaxos-kernel 子仓定义，涵盖调度器相关错误：

| 错误码 | 宏名称 | 含义 | 典型场景 |
|--------|--------|------|----------|
| -200 | -E_SCHED_CLASS_INVALID | 调度类无效 | 指定的调度类不存在 |
| -201 | -E_SCHED_POLICY_INVALID | 调度策略无效 | 策略参数不合法 |
| -202 | -E_SCHED_PRIORITY_INVALID | 优先级无效 | 优先级超出范围 |
| -203 | -E_SCHED_AGENT_LOAD_FAIL | SCHED_AGENT 加载失败 | eBPF 调度器加载失败 |
| -204 | -E_SCHED_AGENT_UNLOAD_FAIL | SCHED_AGENT 卸载失败 | eBPF 调度器卸载失败 |
| -205 | -E_SCHED_AGENT_ATTACH_FAIL | SCHED_AGENT 附加失败 | 调度器附加到 CPU 失败 |
| -206 | -E_SCHED_AGENT_DETACH_FAIL | SCHED_AGENT 分离失败 | 调度器从 CPU 分离失败 |
| -207 | -E_SCHED_AGENT_VERIFY_FAIL | SCHED_AGENT 验证失败 | eBPF 程序验证失败 |
| -208 | -E_SCHED_CPU_OFFLINE | CPU 离线 | 目标 CPU 不可用 |
| -209 | -E_SCHED_CPU_HOTPLUG_FAIL | CPU 热插拔失败 | CPU 热插拔操作失败 |
| -210 | -E_SCHED_AFFINITY_INVALID | CPU 亲和性无效 | 亲和性掩码无效 |
| -211 | -E_SCHED_CGROUP_INVALID | cgroup 无效 | cgroup 配置无效 |
| -212 | -E_SCHED_DEADLINE_MISSED | 截止时间错过 | 实时任务错过截止时间 |
| -213 | -E_SCHED_BANDWIDTH_EXCEEDED | 带宽超限 | 调度带宽超过限制 |
| -214 | -E_SCHED_THROTTLE | 调度限流 | 任务被调度器限流 |
| -215 | -E_SCHED_EEVDF_ERROR | EEVDF 调度器错误 | EEVDF 内部错误 |
| -216 | -E_SCHED_PREEMPT_FAIL | 抢占失败 | 抢占操作失败 |
| -217 | -E_SCHED_MIGRATE_FAIL | 迁移失败 | 任务迁移到其他 CPU 失败 |
| -218 | -E_SCHED_STATS_CORRUPT | 调度统计损坏 | 调度统计数据异常 |
| -219 | -E_SCHED_IDLE_INJECT_FAIL | 空闲时间注入失败 | idle injection 失败 |

#### 2.2.4 IPC 错误码（-300 ~ -399）

IPC 错误码由 airymaxos-kernel 和 airymaxos-services 联合定义：

| 错误码 | 宏名称 | 含义 | 典型场景 |
|--------|--------|------|----------|
| -300 | -E_IPC_MSG_TOO_LARGE | 消息过大 | 消息超出最大长度 |
| -301 | -E_IPC_MSG_TOO_SMALL | 消息过小 | 消息小于最小长度 |
| -302 | -E_IPC_MSG_HDR_INVALID | 消息头无效 | 128B 消息头格式错误 |
| -303 | -E_IPC_MSG_HDR_MAGIC_INVALID | 魔数无效 | magic 不是 0x41524531 |
| -304 | -E_IPC_MSG_HDR_VERSION_INVALID | 版本无效 | 协议版本不兼容 |
| -305 | -E_IPC_MSG_PAYLOAD_INVALID | payload 无效 | payload 协议类型无效 |
| -306 | -E_IPC_MSG_TRUNCATED | 消息截断 | 消息传输不完整 |
| -307 | -E_IPC_MSG_CORRUPT | 消息损坏 | 消息校验和错误 |
| -308 | -E_IPC_CHANNEL_NOT_FOUND | 通道未找到 | 目标 IPC 通道不存在 |
| -309 | -E_IPC_CHANNEL_FULL | 通道已满 | IPC 通道缓冲区满 |
| -310 | -E_IPC_CHANNEL_CLOSED | 通道已关闭 | 远端已关闭通道 |
| -311 | -E_IPC_CHANNEL_TIMEOUT | 通道超时 | 通道操作超时 |
| -312 | -E_IPC_CHANNEL_PERMISSION | 通道权限不足 | 无权限访问目标通道 |
| -313 | -E_IPC_RING_BUFFER_FULL | 环形缓冲区满 | io_uring 提交队列满 |
| -314 | -E_IPC_RING_SETUP_FAIL | 环形缓冲区创建失败 | io_uring 初始化失败 |
| -315 | -E_IPC_RING_CQ_OVERFLOW | 完成队列溢出 | io_uring 完成队列溢出 |
| -316 | -E_IPC_REGISTER_FAIL | 注册失败 | io_uring 注册操作失败 |
| -317 | -E_IPC_ZEROCOPY_FAIL | 零拷贝失败 | 零拷贝操作失败 |
| -318 | -E_IPC_FIXED_BUF_INVALID | 固定缓冲区无效 | 固定缓冲区索引无效 |
| -319 | -E_IPC_MULTISHOT_FAIL | 多重触发失败 | multishot 操作失败 |

#### 2.2.5 内存错误码（-400 ~ -499）

内存错误码由 airymaxos-memory 子仓定义：

| 错误码 | 宏名称 | 含义 | 典型场景 |
|--------|--------|------|----------|
| -400 | -E_MEM_L1_WRITE_FAIL | L1 原始卷写入失败 | 原始记忆写入错误 |
| -401 | -E_MEM_L1_READ_FAIL | L1 原始卷读取失败 | 原始记忆读取错误 |
| -402 | -E_MEM_L1_CORRUPT | L1 原始卷损坏 | 原始数据完整性校验失败 |
| -403 | -E_MEM_L2_INDEX_FAIL | L2 特征层索引失败 | 语义向量索引创建失败 |
| -404 | -E_MEM_L2_QUERY_FAIL | L2 特征层查询失败 | 语义向量检索失败 |
| -405 | -E_MEM_L3_GRAPH_FAIL | L3 结构层图操作失败 | 关系图构建/查询失败 |
| -406 | -E_MEM_L3_BIND_FAIL | L3 绑定算子失败 | 绑定算子执行失败 |
| -407 | -E_MEM_L4_HOMOLOGY_FAIL | L4 持久同调失败 | 同调计算失败 |
| -408 | -E_MEM_L4_RULE_EXTRACT_FAIL | L4 规则提取失败 | 可复用规则提取失败 |
| -409 | -E_MEM_CXL_POOL_FAIL | CXL 内存池化失败 | CXL 跨节点分配失败 |
| -410 | -E_MEM_CXL_NODE_OFFLINE | CXL 节点离线 | CXL 内存节点不可用 |
| -411 | -E_MEM_PMEM_PERSIST_FAIL | PMEM 持久化失败 | 持久化内存写入失败 |
| -412 | -E_MEM_PMEM_RECOVER_FAIL | PMEM 恢复失败 | 持久化内存恢复失败 |
| -413 | -E_MEM_MGLRU_PROMOTE_FAIL | MGLRU 晋升失败 | 冷热数据晋升操作失败 |
| -414 | -E_MEM_MGLRU_DEMOTE_FAIL | MGLRU 降级失败 | 冷热数据降级操作失败 |
| -415 | -E_MEM_FORGET_FAIL | 遗忘操作失败 | 记忆遗忘策略执行失败 |
| -416 | -E_MEM_RECALL_FAIL | 回忆操作失败 | 记忆检索操作失败 |
| -417 | -E_MEM_ARCHIVE_FULL | 归档存储满 | 记忆归档空间不足 |
| -418 | -E_MEM_SEGMENT_LIMIT | 分段限制 | 超出记忆分段限制 |
| -419 | -E_MEM_MIGRATION_FAIL | 迁移失败 | 跨节点记忆迁移失败 |

#### 2.2.6 安全错误码（-500 ~ -599）

安全错误码由 airymaxos-security 子仓定义：

| 错误码 | 宏名称 | 含义 | 典型场景 |
|--------|--------|------|----------|
| -500 | -E_SEC_CAPABILITY_DENIED | capability 拒绝 | 无所需 capability |
| -501 | -E_SEC_CAPABILITY_INVALID | capability 无效 | 令牌格式或内容无效 |
| -502 | -E_SEC_CAPABILITY_EXPIRED | capability 过期 | 令牌已过期 |
| -503 | -E_SEC_CAPABILITY_REVOKED | capability 已撤销 | 令牌已被撤销 |
| -504 | -E_SEC_CAPABILITY_DELEGATE_FAIL | capability 委托失败 | 权限委托操作失败 |
| -505 | -E_SEC_CAPABILITY_RESTRICT_FAIL | capability 限制失败 | 权限限制操作失败 |
| -506 | -E_SEC_LSM_DENIED | LSM 拒绝 | SELinux 或其他 LSM 拒绝 |
| -507 | -E_SEC_LSM_POLICY_INVALID | LSM 策略无效 | SELinux 策略加载失败 |
| -508 | -E_SEC_LSM_CONTEXT_INVALID | LSM 上下文无效 | 安全上下文无效 |
| -509 | -E_SEC_AUDIT_LOG_FAIL | 审计日志失败 | 审计日志写入失败 |
| -510 | -E_SEC_AUDIT_CHAIN_BROKEN | 审计哈希链断裂 | 哈希链完整性验证失败 |
| -511 | -E_SEC_SM2_SIGN_FAIL | SM2 签名失败 | SM2 签名操作失败 |
| -512 | -E_SEC_SM2_VERIFY_FAIL | SM2 验证失败 | SM2 签名验证失败 |
| -513 | -E_SEC_SM3_HASH_FAIL | SM3 哈希失败 | SM3 哈希计算失败 |
| -514 | -E_SEC_SM4_ENCRYPT_FAIL | SM4 加密失败 | SM4 加密操作失败 |
| -515 | -E_SEC_SM4_DECRYPT_FAIL | SM4 解密失败 | SM4 解密操作失败 |
| -516 | -E_SEC_TEE_INIT_FAIL | TEE 初始化失败 | 可信执行环境初始化失败 |
| -517 | -E_SEC_TEE_ATTEST_FAIL | TEE 远程证明失败 | 远程证明验证失败 |
| -518 | -E_SEC_SECCOMP_DENIED | Seccomp 拒绝 | 系统调用被 Seccomp 过滤 |
| -519 | -E_SEC_CFI_VIOLATION | CFI 违规 | 控制流完整性检查失败 |

#### 2.2.7 认知错误码（-600 ~ -699）

认知错误码由 airymaxos-cognition 子仓定义：

| 错误码 | 宏名称 | 含义 | 典型场景 |
|--------|--------|------|----------|
| -600 | -E_COG_CORELOOP_INIT_FAIL | CoreLoop 初始化失败 | kthread 创建失败 |
| -601 | -E_COG_CORELOOP_TIMEOUT | CoreLoop 超时 | 认知循环超时 |
| -602 | -E_COG_SYSTEM1_FAIL | System 1 处理失败 | 辅模型快速分类失败 |
| -603 | -E_COG_SYSTEM2_FAIL | System 2 处理失败 | 主模型深度规划失败 |
| -604 | -E_COG_SWITCH_THRESHOLD | 切换阈值触发 | 双系统切换阈值触发 |
| -605 | -E_COG_PLAN_DAG_FAIL | 规划 DAG 生成失败 | 任务 DAG 构建失败 |
| -606 | -E_COG_PLAN_EXPAND_FAIL | 规划增量扩展失败 | DAG 增量扩展失败 |
| -607 | -E_COG_PLAN_BACKTRACK_FAIL | 规划回退失败 | 智能回退操作失败 |
| -608 | -E_COG_PLAN_DEPTH_EXCEEDED | 规划深度超限 | 超出最大规划深度 |
| -609 | -E_COG_PLAN_BACKTRACK_LIMIT | 回退次数超限 | 回退次数超过 3 次 |
| -610 | -E_COG_LLM_INVOKE_FAIL | LLM 调用失败 | 大语言模型调用失败 |
| -611 | -E_COG_LLM_TOKEN_EXCEEDED | Token 超限 | Token 预算超出限制 |
| -612 | -E_COG_LLM_TIMEOUT | LLM 超时 | LLM 调用超时 |
| -613 | -E_COG_LLM_RESPONSE_INVALID | LLM 响应无效 | LLM 返回格式错误 |
| -614 | -E_COG_WASM_INIT_FAIL | Wasm 初始化失败 | Wasm 运行时初始化失败 |
| -615 | -E_COG_WASM_COMPILE_FAIL | Wasm 编译失败 | Wasm 模块编译失败 |
| -616 | -E_COG_WASM_EXEC_FAIL | Wasm 执行失败 | Wasm 模块执行错误 |
| -617 | -E_COG_WASM_SANDBOX_BREACH | Wasm 沙箱越界 | 沙箱安全边界违规 |
| -618 | -E_COG_FEEDBACK_LOOP_FAIL | 反馈闭环失败 | 反馈机制执行失败 |
| -619 | -E_COG_AGENT_HANG | Agent 挂起 | Agent 执行超时或死锁 |

#### 2.2.8 驱动错误码（-700 ~ -799）

驱动错误码由 airymaxos-kernel 和 airymaxos-services 联合定义：

| 错误码 | 宏名称 | 含义 | 典型场景 |
|--------|--------|------|----------|
| -700 | -E_DRV_PROBE_FAIL | 驱动探测失败 | 设备探测失败 |
| -701 | -E_DRV_INIT_FAIL | 驱动初始化失败 | 驱动程序初始化失败 |
| -702 | -E_DRV_ATTACH_FAIL | 驱动附加失败 | 驱动绑定到设备失败 |
| -703 | -E_DRV_DETACH_FAIL | 驱动分离失败 | 驱动从设备解绑失败 |
| -704 | -E_DRV_DEVICE_NOT_FOUND | 设备未找到 | 目标设备不存在 |
| -705 | -E_DRV_DEVICE_BUSY | 设备忙 | 设备已被其他驱动占用 |
| -706 | -E_DRV_DEVICE_OFFLINE | 设备离线 | 设备热移除或故障 |
| -707 | -E_DRV_IO_FAIL | 设备 I/O 失败 | 设备读写错误 |
| -708 | -E_DRV_DMA_FAIL | DMA 操作失败 | DMA 传输失败 |
| -709 | -E_DRV_IRQ_FAIL | 中断处理失败 | 中断注册或处理失败 |
| -710 | -E_DRV_MMIO_FAIL | MMIO 失败 | 内存映射 I/O 失败 |
| -711 | -E_DRV_VFIO_FAIL | VFIO 失败 | 用户态驱动 VFIO 操作失败 |
| -712 | -E_DRV_VFIO_GROUP_FAIL | VFIO 组失败 | VFIO 组操作失败 |
| -713 | -E_DRV_VFIO_IOMMU_FAIL | VFIO IOMMU 失败 | IOMMU 映射失败 |
| -714 | -E_DRV_DPDK_FAIL | DPDK 失败 | 用户态网络 DPDK 操作失败 |
| -715 | -E_DRV_AF_XDP_FAIL | AF_XDP 失败 | AF_XDP 套接字操作失败 |
| -716 | -E_DRV_FIRMWARE_LOAD_FAIL | 固件加载失败 | 固件文件加载失败 |
| -717 | -E_DRV_FIRMWARE_VERIFY_FAIL | 固件验证失败 | 固件签名验证失败 |
| -718 | -E_DRV_POWER_STATE_FAIL | 电源状态切换失败 | 设备电源管理失败 |
| -719 | -E_DRV_RESET_FAIL | 设备重置失败 | 设备重置操作失败 |

#### 2.2.9 网络错误码（-800 ~ -899）

网络错误码由 airymaxos-cloudnative 子仓定义：

| 错误码 | 宏名称 | 含义 | 典型场景 |
|--------|--------|------|----------|
| -800 | -E_NET_CNI_PLUGIN_FAIL | CNI 插件失败 | 容器网络插件错误 |
| -801 | -E_NET_CNI_IPAM_FAIL | CNI IPAM 失败 | IP 地址管理失败 |
| -802 | -E_NET_CNI_INTERFACE_FAIL | CNI 接口失败 | 网络接口创建失败 |
| -803 | -E_NET_K8S_CRD_APPLY_FAIL | K8s CRD 应用失败 | 自定义资源创建失败 |
| -804 | -E_NET_K8S_CRD_DELETE_FAIL | K8s CRD 删除失败 | 自定义资源删除失败 |
| -805 | -E_NET_K8S_CRD_INVALID | K8s CRD 无效 | CRD 格式或内容无效 |
| -806 | -E_NET_CONTAINERD_SHIM_FAIL | containerd shim 失败 | 容器运行时 shim 失败 |
| -807 | -E_NET_CONTAINERD_IMAGE_FAIL | containerd 镜像失败 | 镜像拉取或推送失败 |
| -808 | -E_NET_OCI_SPEC_INVALID | OCI 规格无效 | 容器规格定义无效 |
| -809 | -E_NET_AGENTCTL_FAIL | agentctl 失败 | agentctl 命令执行失败 |
| -810 | -E_NET_AGENTCTL_INVALID_CMD | agentctl 命令无效 | 不支持的 agentctl 命令 |
| -811 | -E_NET_SUPERNODE_FAIL | 超节点 OS 失败 | 超节点操作失败 |
| -812 | -E_NET_SUPERNODE_UMDK_FAIL | UMDK 失败 | 异构互联底座失败 |
| -813 | -E_NET_SUPERNODE_LATENCY | 超节点延迟超限 | RPC 延迟超出阈值 |
| -814 | -E_NET_EBPF_PROG_LOAD_FAIL | eBPF 程序加载失败 | 网络 eBPF 程序加载失败 |
| -815 | -E_NET_EBPF_MAP_FULL | eBPF 映射表满 | BPF 映射表空间不足 |
| -816 | -E_NET_EBPF_VERIFY_FAIL | eBPF 验证失败 | 网络 eBPF 程序验证失败 |
| -817 | -E_NET_SERVICE_MESH_FAIL | 服务网格失败 | 微服务通信失败 |
| -818 | -E_NET_INGRESS_FAIL | 入口失败 | 流量入口处理失败 |
| -819 | -E_NET_EGRESS_FAIL | 出口失败 | 流量出口处理失败 |

### 2.3 用户态错误码：十六进制体系

用户态错误码采用十六进制表示，便于按子仓和错误类别进行位操作分类。格式为 `0xSSCC0000`：

| 位段 | 含义 | 说明 |
|------|------|------|
| SS (bits 31-24) | 子仓标识 | 0x00=通用, 0x01=services, 0x03=security, 0x04=memory, 0x05=cognition, 0x06=cloudnative, 0x07=system, 0x08=tests |
| CC (bits 23-16) | 错误类别 | 0x01=参数错误, 0x02=资源错误, 0x03=权限错误, 0x04=超时错误, 0x05=状态错误, 0x06=协议错误, 0x07=内部错误, 0x08=外部错误 |
| XXXX (bits 15-0) | 具体错误码 | 每个类别 65536 个具体错误码 |

#### 2.3.1 通用用户态错误码（0x00XX0000）

| 错误码 | 宏名称 | 含义 |
|--------|--------|------|
| 0x00010000 | AGENTRT_ERR_INVALID_PARAM | 参数无效 |
| 0x00020000 | AGENTRT_ERR_OUT_OF_MEMORY | 内存不足 |
| 0x00030000 | AGENTRT_ERR_PERMISSION_DENIED | 权限不足 |
| 0x00040000 | AGENTRT_ERR_TIMEOUT | 操作超时 |
| 0x00050000 | AGENTRT_ERR_INVALID_STATE | 状态无效 |
| 0x00060000 | AGENTRT_ERR_PROTOCOL_ERROR | 协议错误 |
| 0x00070000 | AGENTRT_ERR_INTERNAL_ERROR | 内部错误 |
| 0x00080000 | AGENTRT_ERR_EXTERNAL_ERROR | 外部错误 |

#### 2.3.2 子仓用户态错误码段

| 子仓前缀 | 子仓名称 | 错误码段 | 说明 |
|----------|----------|----------|------|
| 0x01 | airymaxos-services | 0x01010000 ~ 0x01FF0000 | 用户态服务错误 |
| 0x03 | airymaxos-security | 0x03010000 ~ 0x03FF0000 | 安全模块错误 |
| 0x04 | airymaxos-memory | 0x04010000 ~ 0x04FF0000 | 记忆子系统错误 |
| 0x05 | airymaxos-cognition | 0x05010000 ~ 0x05FF0000 | 认知运行时错误 |
| 0x06 | airymaxos-cloudnative | 0x06010000 ~ 0x06FF0000 | 云原生组件错误 |
| 0x07 | airymaxos-system | 0x07010000 ~ 0x07FF0000 | 系统工具错误 |
| 0x08 | airymaxos-tests-linux | 0x08010000 ~ 0x08FF0000 | 测试框架错误 |

---

## 3. 与 agentrt 错误码的映射关系

### 3.1 IRON-9 v2 三层映射

根据 IRON-9 v2 工程铁律，agentrt-linux 与 agentrt 的错误码映射遵循三层架构：

#### 3.1.1 [SC] 共享契约层 — 错误码内嵌定义

错误码定义内嵌在 `include/airymax/` 的 6 个 [SC] 共享契约层头文件中，agentrt 和 agentrt-linux 使用相同的错误码数值和语义：

| [SC] 头文件 | 错误码分类 | 说明 |
|------------|-----------|------|
| `ipc.h` | IPC 错误码 | IPC 相关错误码（消息头校验、payload 类型、通道状态） |
| `security_types.h` | capability 错误码 | capability 相关错误码（令牌无效、权限不足、撤销失败） |
| `memory_types.h` | 记忆错误码 | MemoryRovol 相关错误码（快照失败、层级越界、迁移超时） |
| `sched.h` | 调度错误码 | 调度相关错误码（策略无效、优先级越界、vtime 溢出） |
| `bpf_struct_ops.h` | BPF 调度器错误码 | sched_ext 注册相关错误码 |
| `cognition_types.h` | 认知错误码 | CoreLoopThree 相关错误码（阶段无效、kthread 注册失败） |

通用错误码（`AGENTRT_E*`）属于 [SS] 语义同源层，agentrt 和 agentrt-linux 使用相同的错误码前缀和语义，但具体数值实现各自独立（详见 3.1.2 节）。

#### 3.1.2 [SS] 语义同源层 — 语义一致，数值独立

以下错误码语义与 agentrt 一致，但具体数值可能不同：

| 语义域 | agentrt 实现 | agentrt-linux 实现 | 说明 |
|--------|--------------|---------------------|------|
| 调度错误 | atoms/corekern 调度错误码 | airymaxos-kernel 调度错误码 (-200~-299) | 调度语义一致，数值独立 |
| 安全错误 | cupolas 安全错误码 | airymaxos-security 安全错误码 (-500~-599) | 安全模型一致，数值独立 |
| 认知错误 | coreloopthree 认知错误码 | airymaxos-cognition 认知错误码 (-600~-699) | 认知循环一致，数值独立 |

#### 3.1.3 [IND] 完全独立层 — 无映射

以下错误码完全独立，无映射关系：

| agentrt 独立域 | agentrt-linux 独立域 | 说明 |
|----------------|----------------------|------|
| 平台适配层错误码 | 驱动框架错误码 (-700~-799) | agentrt 无设备驱动概念 |
| 跨平台兼容层错误码 | 网络云原生错误码 (-800~-899) | agentrt 无 K8s/containerd 概念 |
| 构建系统错误码 | 系统级错误码 (-100~-199) | agentrt 无包管理概念 |

### 3.2 错误码转换规则

当 agentrt 在 agentrt-linux 上运行时，错误码转换遵循以下规则：

```
[SC] 层错误码：直接透传，无需转换
[SS] 层错误码：通过映射表转换（agentrt 错误码 → agentrt-linux 错误码）
[IND] 层错误码：不转换，各自独立处理
```

### 3.3 错误码双向映射表（[SS] 层）

| agentrt 错误码 | 语义 | agentrt-linux 错误码 | 转换方向 |
|----------------|------|----------------------|----------|
| CORESCHED_ERR_INVALID | 调度参数无效 | -E_SCHED_CLASS_INVALID (-200) | 双向 |
| CORESCHED_ERR_TIMEOUT | 调度超时 | -E_SCHED_DEADLINE_MISSED (-212) | 双向 |
| CUPOLAS_ERR_DENIED | 权限拒绝 | -E_SEC_CAPABILITY_DENIED (-500) | 双向 |
| CUPOLAS_ERR_INVALID_CAP | 无效 capability | -E_SEC_CAPABILITY_INVALID (-501) | 双向 |
| LOOP3_ERR_TIMEOUT | 认知循环超时 | -E_COG_CORELOOP_TIMEOUT (-601) | 双向 |
| LOOP3_ERR_PLAN_FAIL | 规划失败 | -E_COG_PLAN_DAG_FAIL (-605) | 双向 |
| MEMROVOL_ERR_WRITE | 记忆写入失败 | -E_MEM_L1_WRITE_FAIL (-400) | 双向 |
| MEMROVOL_ERR_READ | 记忆读取失败 | -E_MEM_L1_READ_FAIL (-401) | 双向 |

---

## 4. 错误码使用规范

### 4.1 错误码定义规范

所有错误码定义必须遵循以下规范：

#### 4.1.1 内核态错误码定义模板

```c
/**
 * @brief 错误码宏定义 — 内核态
 * @file include/airymax/airymax_kernel_errno.h
 * @since 1.0.1
 */

/* === 通用错误码（复用 Linux 内核 errno） === */
/* 直接使用 Linux 内核标准 errno，无需重新定义 */

/* === 调度错误码（-200 ~ -299） === */
#define E_SCHED_CLASS_INVALID        (-200)  /**< 调度类无效 */
#define E_SCHED_POLICY_INVALID       (-201)  /**< 调度策略无效 */
#define E_SCHED_PRIORITY_INVALID     (-202)  /**< 优先级无效 */
#define E_SCHED_AGENT_LOAD_FAIL      (-203)  /**< SCHED_AGENT 加载失败 */
#define E_SCHED_AGENT_UNLOAD_FAIL    (-204)  /**< SCHED_AGENT 卸载失败 */

/* === IPC 错误码（-300 ~ -399） === */
#define E_IPC_MSG_TOO_LARGE          (-300)  /**< 消息过大 */
#define E_IPC_MSG_TOO_SMALL          (-301)  /**< 消息过小 */
#define E_IPC_MSG_HDR_INVALID        (-302)  /**< 消息头无效 */
#define E_IPC_MSG_HDR_MAGIC_INVALID  (-303)  /**< 魔数无效 */

/* 更多错误码定义见上文完整列表 */
```

#### 4.1.2 用户态错误码定义模板

```c
/**
 * @brief 错误码宏定义 — 用户态
 * @file include/airymax/ipc.h（[SC] 共享契约层）
 * @since 1.0.1
 */

/* === 通用用户态错误码（0x00XX0000） === */
#define AGENTRT_ERR_INVALID_PARAM    0x00010000  /**< 参数无效 */
#define AGENTRT_ERR_OUT_OF_MEMORY    0x00020000  /**< 内存不足 */
#define AGENTRT_ERR_PERMISSION_DENIED 0x00030000 /**< 权限不足 */
#define AGENTRT_ERR_TIMEOUT          0x00040000  /**< 操作超时 */
#define AGENTRT_ERR_INVALID_STATE    0x00050000  /**< 状态无效 */
#define AGENTRT_ERR_PROTOCOL_ERROR   0x00060000  /**< 协议错误 */
#define AGENTRT_ERR_INTERNAL_ERROR   0x00070000  /**< 内部错误 */
#define AGENTRT_ERR_EXTERNAL_ERROR   0x00080000  /**< 外部错误 */

/* === 子仓错误码基址 === */
#define AGENTRT_ERR_SERVICES_BASE    0x01000000  /**< services 子仓错误码基址 */
#define AGENTRT_ERR_SECURITY_BASE    0x03000000  /**< security 子仓错误码基址 */
#define AGENTRT_ERR_MEMORY_BASE      0x04000000  /**< memory 子仓错误码基址 */
#define AGENTRT_ERR_COGNITION_BASE   0x05000000  /**< cognition 子仓错误码基址 */
#define AGENTRT_ERR_CLOUDNATIVE_BASE 0x06000000  /**< cloudnative 子仓错误码基址 */
#define AGENTRT_ERR_SYSTEM_BASE      0x07000000  /**< system 子仓错误码基址 */
#define AGENTRT_ERR_TESTS_BASE       0x08000000  /**< tests-linux 子仓错误码基址 */
```

### 4.2 错误码传播规范

错误码在系统中的传播必须遵循以下规则：

#### 4.2.1 传播方向

```
内核态（负整数） ─── syscall 返回 ───> 用户态（十六进制）
     ↑                                      │
     │                                      │
     └── 用户态调用 syscall 时传入错误码 ──────┘
```

#### 4.2.2 传播规则

| 规则 | 说明 | 示例 |
|------|------|------|
| **错误不丢失** | 每一层向上传播时必须保留原始错误码 | 使用 `agentrt_error_wrap()` 包装 |
| **错误可丰富** | 每一层可以添加本层上下文，但不修改原始错误码 | 添加模块名、函数名、行号 |
| **错误不吞没** | 禁止在中间层吞没错误（返回 OK 掩盖错误） | 禁止 `if (err) return OK;` |
| **错误链可追溯** | 通过 `agentrt_error_chain` 结构体维护完整错误链 | 类似 `errno` 但更丰富 |

#### 4.2.3 错误包装示例

```c
/**
 * @brief 错误包装函数
 * @param err 原始错误码（内核态负整数）
 * @param module 模块名
 * @param func 函数名
 * @param line 行号
 * @return 带上下文的错误码（用户态十六进制）
 */
uint32_t agentrt_error_wrap(int err, const char *module,
                            const char *func, int line);

/* 使用示例 */
int ret = io_uring_submit(&ring);
if (ret < 0) {
    return agentrt_error_wrap(ret, "ipc_svc", __func__, __LINE__);
}
```

### 4.3 错误码日志规范

错误码日志必须包含以下信息，确保错误可追溯（E-6 原则）：

| 字段 | 说明 | 是否必需 |
|------|------|----------|
| error_code | 错误码数值 | 必需 |
| error_name | 错误码宏名称 | 必需 |
| human_readable | 人类可读的错误描述 | 必需 |
| context | 错误上下文（键值对） | 必需 |
| suggestion | 建议的修复操作 | 推荐 |
| trace_id | 分布式追踪 ID | 推荐 |
| timestamp | 时间戳 | 必需 |
| module | 产生错误的模块 | 必需 |

**日志格式模板**：

```
[ERROR] <error_code> (<error_name>): <human_readable>
  context: module=<module>, func=<func>, line=<line>, trace_id=<trace_id>
  suggestion: <action>
```

---

## 5. 错误码注册与维护流程

### 5.1 新增错误码流程

新增错误码必须遵循以下流程：

```mermaid
graph TD
    A[确定错误场景] --> B{是否可用现有错误码?}
    B -->|是| C[使用现有错误码]
    B -->|否| D[选择合适的分段区间]
    D --> E[在对应 [SC] 头文件中定义]
    E --> F[更新 error_code_reference.md]
    F --> G[在相关子仓代码中使用]
    G --> H[添加单元测试]
    H --> I[提交 PR 并通过审查]
    I --> J[合并并更新版本号]

    C --> G

    classDef start fill:#43a047,stroke:#1b5e20,color:#ffffff
    classDef decision fill:#fb8c00,stroke:#e65100,color:#ffffff
    classDef action fill:#2196f3,stroke:#0d47a1,color:#ffffff
    classDef endNode fill:#9e9e9e,stroke:#424242,color:#ffffff

    class A start
    class B decision
    class C,D,E,F,G,H,I action
    class J endNode
```

### 5.2 错误码变更流程

| 变更类型 | 处理方式 | 审查要求 |
|----------|----------|----------|
| **新增错误码** | 在对应分段内分配未使用的数值 | 子仓治理组审查 |
| **修改错误码语义** | 标记 @deprecated 旧定义，新增新定义 | 工程规范委员会审查 + ADR 记录 |
| **删除错误码** | 标记 @deprecated，保留至少 2 个版本周期 | 工程规范委员会审查 + 公告 |
| **调整分段区间** | 必须通过 ADR 评审 | 工程规范委员会投票 |
| **修改 [SC] 层错误码** | 必须 agentrt + agentrt-linux 双向同步 | 工程规范委员会联合审查 |

### 5.3 错误码弃用流程

```
1. 在对应 [SC] 头文件中添加 @deprecated 注释，并注明替代错误码
2. 在 error_code_reference.md 中标记弃用状态
3. 在代码中移除所有使用旧错误码的引用
4. 发布弃用公告（至少提前 2 个版本周期）
5. 在 2 个版本周期后正式删除
```

### 5.4 错误码维护责任

| 分段 | 维护主体 | 审查主体 |
|------|----------|----------|
| 通用错误码（-1~-99） | 全部子仓 | 工程规范委员会 |
| 系统错误码（-100~-199） | airymaxos-system 治理组 | Base Systems 治理组 |
| 调度错误码（-200~-299） | airymaxos-kernel 治理组 | Kernel 治理组 |
| IPC 错误码（-300~-399） | airymaxos-kernel + airymaxos-services 治理组 | Kernel + Base Systems 治理组 |
| 内存错误码（-400~-499） | airymaxos-memory 治理组 | Memory 治理组 |
| 安全错误码（-500~-599） | airymaxos-security 治理组 | Security 治理组 |
| 认知错误码（-600~-699） | airymaxos-cognition 治理组 | Cognition 治理组 |
| 驱动错误码（-700~-799） | airymaxos-kernel + airymaxos-services 治理组 | Kernel + Base Systems 治理组 |
| 网络错误码（-800~-899） | airymaxos-cloudnative 治理组 | Cloud Native 治理组 |
| 用户态错误码（0x00XX0000~） | 各子仓治理组 | 对应治理组 |

---

## 6. 错误码验证与测试

### 6.1 错误码验证检查

| 检查项 | 工具 | 频率 |
|--------|------|------|
| 错误码未注册 | 静态分析脚本扫描 | 每次 PR |
| 错误码冲突 | 自动化冲突检测 | 每次 PR |
| [SC] 层错误码一致性 | 跨仓 CI 双向校验 | 每次 PR |
| 错误码使用规范 | 代码审查 + 静态分析 | 每次 PR |
| 错误码弃用残留 | Grep 扫描 | 每次发布 |
| 错误码测试覆盖 | 单元测试覆盖率检查 | 每次 PR |

### 6.2 错误码测试用例

每个错误码必须有对应的测试用例，验证以下场景：

1. **触发场景**：验证错误码能被正确触发
2. **传播路径**：验证错误码在跨层传播中不丢失
3. **日志输出**：验证错误日志包含完整上下文
4. **恢复路径**：验证错误恢复路径正确处理该错误码
5. **映射正确性**：验证 [SS] 层错误码映射正确

---

## 7. 相关文档

- [项目管理规范总览](README.md)：项目管理规范顶层入口
- [SBOM 规范](SBOM.md)：软件物料清单规范
- [集成标准总览](../40-integration/README.md)：集成标准
- [agentrt 集成规范](../40-integration/agentrt_integration.md)：与 agentrt 的集成规范
- [接口设计](../../30-interfaces/README.md)：系统调用与 IPC 接口
- [五维正交原则](../../10-architecture/02-five-dimensional-principles.md)：E-6 错误可追溯原则
- IRON-9 v2 工程铁律（闭源内部参考）

---

## 8. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.1.1 | 2026-07-07 | 初始版本（双错误码体系 + 9 段内核态 + 8 子仓用户态 + agentrt 三层映射 + 使用规范 + 维护流程） |
| 1.0.1 | 2027-XX-XX | 首个开发版本（与代码实现同步验证） |