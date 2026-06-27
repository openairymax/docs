# Airymax CI/CD 标准化配置文档

**最新**: 2026-06-09
**状态**: 维护中
**路径**: OpenAirymax/Docs/Capital_Guides/ci_cd_documentation.md
---

## 1. 架构概述

### 1.1 MCIS-based CI/CD 设计原则

| 原则 | 说明 | MCIS维度 |
|------|------|----------|
| **正交分离** | 各模块独立构建、测试、部署 | 五维正交系统 |
| **控制论耦合** | 负反馈(阻断)、正反馈(加速)、前馈(预测) | 控制论机制 |
| **缓存优先** | 三级缓存 (ccache/vcpkg/pip) 减少70%+构建时间 | 记忆体 |
| **容错机制** | 网络重试、降级构建、详细错误报告 | 安全穹顶体 |
| **并行优化** | 多模块矩阵并行构建 | 执行体 |
| **可观测性** | 质量监控、趋势分析、技术债务追踪 | 可观测体 |

## 2. 流水线架构（3个核心工作流）

### 2.1 工作流清单

Airymax CI/CD 由 3 个核心工作流构成（v5.0）：

| 工作流 | 触发条件 | 功能 |
|--------|---------|------|
| `ci.yml` | push/PR main,develop | C核心构建 + CTest + Sanitizer(ASan/LSan/UBSan) + MemoryRovol验证 + BAN反模式检测 + Clang-Tidy + Cppcheck + 质量门禁 + Python检查 + 合约检查 |
| `docker-e2e.yml` | push/PR (paths: deploy/**, sdk/typescript/**) | Docker镜像lint/build/push + TypeScript SDK构建 + Docker服务健康检查 |
| `release.yml` | tag推送(v*) + manual | C核心+SDK打包 + SBOM生成 + 签名 + GitHub Release |

**已移除的冗余工作流**（功能已合并至3个核心工作流）：
- `sdk-ci.yml` → 合并入 ci.yml
- `docs-ci.yml` → 合并入 release.yml
- `protocol-ci.yml` → 合并入 ci.yml (protocol job)
- `dependency-update.yml` → 由 Dependabot 替代
- `quality-monitoring.yml` → 合并入 ci.yml (quality-gate job)
- `stale.yml` → 由 GitHub 内置功能替代
- `stub-check.yml` → 合并入 ci.yml (BAN审计)
- `build-desktop.yml` → 合并入 docker-e2e.yml (desktop-build job)
- `api-docs.yml` → 合并入 release.yml
- `docs-check.yml` → 合并入 release.yml
- `tests-test.yml` → 合并入 ci.yml
- `sanitizer-ci.yml` → 合并入 ci.yml (sanitizer-test job)
- `memoryrovol-integration.yml` → 合并入 ci.yml (memoryrovol-oss/pro jobs)
- `docker-ci.yml` → 合并入 docker-e2e.yml
- `e2e.yml` → 合并入 docker-e2e.yml
- `quality-security.yml` → 合并入 ci.yml (ban-scan + quality-gate jobs)

---

## 2. 文件结构

### 2.1 CI脚本目录 (`scripts/ci/`)

```
scripts/ci/                         # CI/CD 工具集目录
├── ci-run.sh                       # 主CI编排脚本 (6阶段流水线)
├── install-deps.sh                 # 统一依赖安装脚本 (带重试)
├── build-module.sh                 # 模块化构建脚本 (支持增量构建)
├── run-tests.sh                    # 测试执行脚本 (单元/集成/内存检测)
├── quality-gate.sh                 # 质量门禁脚本 (静态分析/格式检查)
├── deploy-artifacts.sh             # 制品打包与部署脚本
├── requirements-linux.txt          # Linux apt 依赖清单
├── requirements-macos.txt          # macOS Homebrew 依赖清单
└── CI_CD_DOCUMENTATION.md          # 本文档
```

### 2.2 工作流目录 (`.github/workflows/`)

```
.github/workflows/                  # GitHub Actions 工作流
├── ci.yml                          # 主CI流水线 (13个Jobs: 构建/测试/Sanitizer/MemoryRovol/质量门禁/...)
├── docker-e2e.yml                  # Docker + E2E流水线 (6个Jobs: lint/build/push/desktop/health)
└── release.yml                     # 发布流水线 (4个Jobs: 打包/SBOM/签名/Release)
```

---

## 3. 依赖管理体系

### 3.1 四层依赖架构

```
Tier 0: Core (CI环境自带，无需安装)
├── CMake >= 3.20
├── GCC >= 9 / Clang >= 10 / MSVC 2019+
├── pkg-config
├── Threads (POSIX/Win32)
└── Git

Tier 1: System Packages (必须，构建阻塞)
├── libcurl (HTTP客户端)
├── libyaml (YAML解析)
├── libcjson (JSON处理)
├── OpenSSL (加密/TLS)
└── 平台特定: pthreads, ws2_32

Tier 2: Specialized (可选，模块依赖)
├── libmicrohttpd ← gateway 模块
├── libwebsockets  ← gateway 模块
├── SQLite3         ← heapstore, atoms 可选
├── libevent         ← atoms 可选
└── tiktoken*       ← commons (需特殊处理*)

Tier 3: Tooling (仅CI使用)
├── cppcheck (静态分析)
├── clang-format (格式化)
├── valgrind (内存检测)
├── gcovr (覆盖率)
└── GTest/CUnit (测试框架)
```

### 3.2 tiktoken 特殊处理方案

**问题**: tiktoken 是 Python 库，没有 C 语言原生接口，但 `agentos/commons/CMakeLists.txt` 通过 `pkg_check_modules(TIKTOKEN REQUIRED tiktoken)` 强制要求。

**解决方案**: 在 CI 中创建 **pkg-config 存根 (Stub)**

```bash
cat > /usr/local/lib/pkgconfig/tiktoken.pc << 'EOF'
prefix=/usr/local
exec_prefix=${prefix}
libdir=${exec_prefix}/lib
includedir=${prefix}/include

Name: tiktoken
Description: Tokenizer library (CI stub for Airymax)
Version: 0.5.2
Libs: -L${libdir} -ltiktoken
Cflags: -I${includedir}
EOF

echo 'void tiktoken_init(void) {}' | gcc -c - -o /tmp/tiktoken_stub.o
ar rcs /usr/local/lib/libtiktoken.a /tmp/tiktoken_stub.o
```

### 3.3 完整依赖矩阵

| 依赖名称 | 类型 | 必需? | 使用模块 | Linux包名 | Homebrew | vcpkg |
|---------|------|-------|---------|-----------|----------|-------|
| Threads | find_package | YES | 全部 | 内置 | 内置 | 内置 |
| PkgConfig | find_package | YES | daemon, commons | pkg-config | pkg-config | - |
| cJSON | pkg_check_modules | YES | daemon, gateway, commons | libcjson-dev | cjson | cjson |
| libcurl | pkg_check_modules | YES | daemon/common, llm_d | libcurl4-openssl-dev | curl | curl |
| yaml/libyaml | pkg_check_modules | YES | commons, daemon | libyaml-dev | yaml-cpp | yaml-cpp |
| OpenSSL | find_package | YES | cupolas | libssl-dev | openssl@3 | openssl |
| tiktoken* | pkg_check_modules | Stub | commons | (Stub) | (Stub) | (Stub) |
| libmicrohttpd | pkg_check_modules | Gateway | libmicrohttpd-dev | libmicrohttpd | libmicrohttpd |
| libwebsockets | pkg_check_modules | Gateway | libwebsockets-dev | libwebsockets | libwebsockets |
| SQLite3 | find_package | Optional | atoms, heapstore | libsqlite3-dev | sqlite | sqlite3 |
| libevent | find_package | Optional | atoms | libevent-dev | libevent | - |

---

## 4. 构建流水线设计

### 4.1 构建命令标准

```bash
# 安装依赖
chmod +x scripts/ci/install-deps.sh
./scripts/ci/install-deps.sh

# 模块化构建 (支持增量构建)
chmod +x scripts/ci/build-module.sh
./scripts/ci/build-module.sh --module <module> --type Release --parallel 4 --incremental

# 测试
chmod +x scripts/ci/run-tests.sh
./scripts/ci/run-tests.sh --module <module> --category all

# 质量门禁
chmod +x scripts/ci/quality-gate.sh
./scripts/ci/quality-gate.sh

# 制品打包
chmod +x scripts/ci/deploy-artifacts.sh
./scripts/ci/deploy-artifacts.sh
```

### 4.2 支持的模块列表

| 模块 | 源码路径 | MCIS维度 | CMake选项 |
|------|---------|----------|-----------|
| daemon | agentos/daemon | 执行体 | -DBUILD_TESTS=ON -DENABLE_LLM_DUMMY=ON |
| atoms | agentos/atoms | 认知体 | -DBUILD_TESTS=ON |
| commons | agentos/commons | 认知体 | -DBUILD_TESTS=ON |
| cupolas | agentos/cupolas | 安全穹顶体 | -DBUILD_TESTS=ON |
| gateway | agentos/gateway | 执行体 | -DBUILD_TESTS=ON |
| heapstore | agentos/heapstore | 记忆体 | -DBUILD_TESTS=ON |
| manager | agentos/manager | 执行体 | -DBUILD_TESTS=ON |

---

## 5. 缓存策略

### 5.1 智能缓存系统

| 缓存类型 | 工具 | Key生成规则 | 大小限制 |
|---------|------|------------|---------|
| C++编译缓存 | ccache | MCIS依赖文件组合哈希 + 每周轮换 | 1GB |
| C++依赖缓存 | vcpkg | vcpkg.json哈希 | 默认 |
| Python依赖缓存 | pip | requirements文件哈希 | 默认 |

### 5.2 预期性能提升

| 操作 | 无缓存 | 有缓存 | 提升 |
|------|--------|--------|------|
| Linux 依赖安装 | ~8 min | ~30 s | 94% |
| C++ 编译 (增量) | 100% | 20-40% | 60-80% |
| 总构建时间 (缓存命中) | ~45 min | ~25 min | 44% |

---

## 6. 错误处理与容错

### 6.1 重试机制

install-deps.sh 中的重试逻辑:
- 最大重试次数: 3
- 重试延迟: 5秒，指数退避 (5s -> 10s -> 20s)
- 网络错误自动重试

### 6.2 降级策略

| 场景 | 主要方案 | 降级方案 |
|------|---------|---------|
| 统一脚本失败 | scripts/ci/install-deps.sh | 直接 apt-get install 最小集 |
| 测试套件失败 | ctest --output-on-failure | 继续后续步骤，标记警告 |
| 可选依赖缺失 | 完整安装 | 跳过，记录 warning |
| 格式检查失败 | 阻塞 PR | 仅报告，不阻塞 |
| 测试文件不存在 | pytest指定文件 | 跳过并发出notice |

---

## 7. 质量门禁

### 7.1 检查项目

| 检查项 | 工具 | 检查范围 | 阻塞? |
|--------|------|---------|-------|
| C/C++静态分析 | cppcheck | agentos/*/src | 否 |
| C/C++格式检查 | clang-format | *.c/*.h | 否 |
| Python质量 | pylint + mypy | agentos/toolkit/python | 否 |
| 内存检测 | Valgrind | daemon (Debug+ASan) | 否 |
| 安全扫描 | Trivy + Gitleaks | 全项目 | 是(严重漏洞) |

---

## 8. MCIS工作流映射

| MCIS维度 | 工作流 | 触发条件 | 控制论机制 |
|----------|--------|---------|-----------|
| 认知体+执行体+记忆体 | ci.yml | push/PR | 负反馈:构建/测试/Sanitizer失败阻断 |
| 安全穹顶体+可观测体 | ci.yml (ban-scan + quality-gate jobs) | push/PR | 负反馈:安全/BAN合规阻断 |
| 容器化部署体 | docker-e2e.yml | push/PR (paths) | 负反馈:构建/健康检查失败阻断 |
| 系统观发布体 | release.yml | tag push | 正反馈:发布加速 |

---

## 9. 维护指南

### 9.1 更新依赖版本

1. 编辑 `scripts/ci/requirements-linux.txt` 或 `scripts/ci/requirements-macos.txt`
2. 编辑 `scripts/development/config/vcpkg.json`
3. 提交代码 -> CI自动验证 -> 合并到main

### 9.2 排查常见问题

| 错误信息 | 可能原因 | 解决方案 |
|---------|---------|---------|
| Could not find TIKTOKEN | 存根未创建 | 检查 tiktoken.pc 是否存在 |
| vcpkg.json not found | 文件缺失 | 确认 scripts/development/config/vcpkg.json 存在 |
| OpenSSL not found | macOS路径问题 | 设置 OPENSSL_ROOT_DIR |
| apt-get timeout | 网络问题 | 重试机制会自动处理 |
| ctest timeout | 测试挂起 | 增加 --timeout 值 |
| Module not defined | build-module.sh缺少定义 | 在MODULE_SOURCES中添加模块 |
| Test file not found | 测试文件不存在 | 工作流已添加文件存在性检查 |

---

## 变更历史

| 版本 | 日期 | 变更内容 |
|------|------|---------|
| v3.0.0 | 2026-04-09 | MCIS理论集成; 更新文件结构; 修复引用一致性; 增量构建支持 |
| v2.2.0 | 2026-04-06 | 目录迁移: ci/ -> scripts/ci/ |
| v2.0.0 | 2026-04-02 | 初始版本，完整CI/CD标准化配置 |

---

*SPHARX Ltd. - Airymax Project*