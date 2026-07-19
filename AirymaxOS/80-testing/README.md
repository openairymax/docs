Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）测试体系设计
> **文档定位**：agentrt-linux（AirymaxOS）测试工程体系主索引（KUnit + kselftest + 集成测试 + CI 流水线）\
> **文档版本**：v1.0\
> **最后更新**：2026-07-17\
> **上级文档**：[AirymaxOS 总览](../README.md)\
> **同源映射**：agentrt 7 层自动化验证 + Linux 6.6 测试框架（KUnit/kselftest/动态分析）\
> **理论根基**：Linux 内核测试体系 + Airymax E-8 可测试性 + A-4 完美主义 + SSoT v2 CI 强制校验

---

## 1. 模块概述

agentrt-linux 测试体系是工程标准可执行性的核心保障。它继承 Linux 内核 30+ 年沉淀的多层测试哲学（KUnit 白盒 + kselftest 系统级 + 动态分析 + 静态分析 + 覆盖度量），并在其上扩展智能体操作系统专属的sched_tac 调度测试、IPC 零拷贝测试、[SC] 头文件逐字节校验、Agent 行为契约测试、模糊测试、形式化验证等。本目录覆盖四方面职责：

1. **KUnit 单元测试**：白盒单元测试，毫秒级执行，验证单函数行为契约（含 A-UEF 错误码、A-ULP 128B 记录格式、A-UCS 配置加载）。
2. **kselftest 系统级测试**：从用户态测试内核特性，重点覆盖sched_tac 调度类（`SCHED_DEADLINE` / `SCHED_FIFO` / `EEVDF`）、io_uring IORING_OP_URING_CMD 零拷贝、纯 C LSM 钩子。
3. **集成测试**：跨子仓跨模块的端到端测试，验证 IRON-9 v3 四层模型的 [SC] 头文件逐字节一致、agentrt ↔ agentrt-linux 同源 API 互操作。
4. **CI 流水线**：`sc-dual-ci.yml`（[SC] 逐字节校验）+ `ssot-validate.yml`（四层模型归属校验）+ 覆盖率门槛（kernel 90% / security 95%，关键路径 100%）。

### 1.1 测试体系分层

| 层级 | 类型 | 工具 | 时间尺度 | 范围 |
|------|------|------|---------|------|
| L1 | 白盒单元测试 | KUnit | 毫秒级 | 单函数 |
| L2 | 系统级测试 | kselftest | 秒级 | 子系统 |
| L3 | 内核自检 | `lib/test_*` | 秒级 | 内核特性 |
| L4 | 动态分析 | KASAN/KFENCE/UBSAN/KCSAN/lockdep/kmemleak | 运行时 | 内存/并发 |
| L5 | 静态分析 | Sparse/Smatch/Coccinelle/clang-analyzer | 编译时 | 代码模式 |
| L6 | 覆盖度量 | KCOV/gcov | 运行时 | 代码覆盖 |
| L7 | ftrace 启动自检 | ftrace + ring buffer | 启动时 | 跟踪 |
| **L8** | **Agent 契约测试** | **agentrt-linux 专属** | **秒级** | **Agent 行为** |
| **L9** | **模糊测试** | **syzkaller + agentrt-linux 扩展** | **小时级** | **输入边界** |
| **L10** | **形式化验证** | **seL4 风格 + TLA+** | **天级** | **关键路径** |

### 1.2 agentrt-linux 专属测试

- **sched_tac 调度测试**：验证 `SCHED_DEADLINE` / `SCHED_FIFO` / `EEVDF` 三层调度类组合的 Agent 8 态生命周期映射（INACTIVE→SPAWNING→READY→RUNNING→BLOCKED→STOPPING→STOPPED→DEAD）
- **IPC 零拷贝测试**：验证 `IORING_OP_URING_CMD` 的 fastpath 性能（~160ns）与 page flipping 路径未启用
- **[SC] 头文件逐字节校验**：`sc-dual-ci.yml` 双端 diff 10 个 `include/uapi/linux/airymax/*.h` 头文件
- **纯 C LSM 验证**：验证 `airy_lsm` 钩子注册、capability 缓存命中、BPF LSM 未启用
- **alloc_pages + mmap 验证**：验证共享内存未使用 `dma_alloc_coherent`

---

## 2. 技术选型声明

agentrt-linux v1.0 测试体系在内核调度、IPC 传输、安全钩子、内存分配与同源代码共享五个维度遵循 [AirymaxOS 总览](../README.md) §2 的不可妥协基线。测试体系是五大选型的**验证守护者**——CI 流水线在每次提交时强制校验五大选型未被偏离，任何回归均阻塞合并。五个维度的选型在本目录的具体落地如下：

| # | 技术维度 | 选定方案 | 明确不采用的方案 | 在本目录的落地 |
|---|---------|---------|----------------|--------------|
| 1 | **内核调度** | **sched_tac**：复用 Linux 6.6 原生 `SCHED_DEADLINE` / `SCHED_FIFO` / `EEVDF` 调度类 | **不使用 sched_ext**（不引入 eBPF 调度器、不使用 `SCHED_EXT=7` 调度类） | kselftest `sched/` 子目录验证sched_tac 三层调度类组合；CI 断言 `CONFIG_SCHED_EXT` 未启用 |
| 2 | **IPC 零拷贝** | **IORING_OP_URING_CMD**：通过 io_uring 命令操作码实现内核↔用户态零拷贝传输 | **不使用 page flipping**（不交换物理页、不破坏内存布局稳定性） | IPC fastpath 性能基准测试（~160ns SLA）；CI 断言 page flipping 代码路径未编译 |
| 3 | **安全钩子** | **纯 C LSM**：以纯 C 实现的 `airy_lsm` 通过 `security_hook_list` 注册 | **不使用 BPF LSM**（不依赖 BPF LSM 框架、不通过 eBPF 程序挂载安全钩子） | KUnit 测试纯 C LSM 钩子注册与 capability 缓存；CI 断言 `CONFIG_BPF_LSM` 未启用 |
| 4 | **内存分配** | **alloc_pages + mmap**：通过 `alloc_pages` 分配物理页后 `vm_map_pages` / `remap_pfn_range` 映射 | **不使用 DMA 一致性内存**（不调用 `dma_alloc_coherent`、不依赖硬件一致性缓存） | KUnit 测试 `alloc_pages + mmap` 内存路径；CI 断言 `dma_alloc_coherent` 未在共享内存路径调用 |
| 5 | **同源代码共享** | **IRON-9 v3 四层模型**：[SC] 共享契约层 + [SS] 语义同源层 + [IND] 独立实现层 + [DSL] 降级生存层 | （v2 三层模型升级为 v3 四层模型，新增 [DSL] 降级生存层） | `sc-dual-ci.yml` 双端逐字节校验 10 个 [SC] 头文件；`ssot-validate.yml` 校验四层归属一致性 |

### 2.1 IRON-9 v3 四层模型在测试体系的归属

| 技术点 | [SC] | [SS] | [IND] | [DSL] | 落地文档 |
|--------|:----:|:----:|:-----:|:-----:|---------|
| [SC] 逐字节校验测试 | ● | — | — | — | [01-kunit-framework.md](01-kunit-framework.md) |
| sched_tac 调度测试 | — | — | ● | — | [02-kselftest.md](02-kselftest.md) |
| IPC 零拷贝性能测试 | — | — | ● | — | [02-kselftest.md](02-kselftest.md) |
| [DSL] 降级块自检 | — | — | — | ● | [01-kunit-framework.md](01-kunit-framework.md) |
| 同源 API 互操作测试 | — | ● | — | — | [01-kunit-framework.md](01-kunit-framework.md) |

---

## 3. 文档索引

本目录现有文档（v1.0 范围）：

```
80-testing/
├── README.md                       # 本文件（v1.0）
├── 01-kunit-framework.md           # KUnit 单元测试框架 + [SC] 契约测试
└── 02-kselftest.md                 # kselftest 系统级测试 + sched_tac 调度测试 + IPC 零拷贝测试
```

| # | 文档 | 版本 | 内容概要 |
|---|------|------|---------|
| — | [README.md](README.md) | v1.0 | 测试体系主索引（本文件） |
| 1 | [01-kunit-framework.md](01-kunit-framework.md) | v1.0 | KUnit 白盒单元测试框架、A-UEF 错误码契约测试、A-ULP 128B 记录格式测试、[SC] 头文件一致性测试、[DSL] 降级块自检 |
| 2 | [02-kselftest.md](02-kselftest.md) | v1.0 | kselftest 系统级测试、sched_tac 调度类测试（`SCHED_DEADLINE` / `SCHED_FIFO` / `EEVDF`）、IPC 零拷贝性能基准（~160ns）、纯 C LSM 钩子验证 |

### 3.1 后续规划文档（1.0.1 版本）

以下文档在 1.0.1 版本完成，不在 v1.0 范围内：

- `03-kernel-selftests.md`：`lib/test_*` 内核自检
- `04-dynamic-analysis.md`：动态分析（KASAN/KFENCE/UBSAN/KCSAN/lockdep）
- `05-static-analysis.md`：静态分析（Sparse/Smatch/Coccinelle）
- `06-coverage-metrics.md`：覆盖度量（KCOV/gcov）+ 覆盖率门槛
- `07-ftrace-selftest.md`：ftrace 启动自检
- `08-agent-contract-testing.md`：agentrt-linux 专属 Agent 行为契约测试
- `09-fuzz-testing.md`：模糊测试（syzkaller + 扩展）
- `10-formal-verification.md`：形式化验证（seL4 风格 + TLA+）

---

## 4. Airymax Unify Design 映射

本目录与 Airymax Unify Design 五模块（A-UEF/A-ULP/A-UCS/A-ULS/A-IPC）的关系详见 [AirymaxOS 总览](../README.md) §5。测试体系主要承载 **sched_tac 调度测试**（A-ULS 调度）、**IPC 零拷贝测试**（A-IPC）、**[SC] 头文件逐字节校验**（IRON-9 v3 [SC] 层），其余模块为辅助关系。

| Unify 模块 | 关系 | 在本目录的体现 |
|-----------|------|--------------|
| **A-UEF** | **核心** | A-UEF 错误码 / Fault 码空间的 KUnit 契约测试；[SC] `error.h` 双端逐字节校验 |
| **A-ULP** | 辅助 | A-ULP 128B 日志记录格式的 KUnit 测试；[SC] `log_types.h` 双端逐字节校验 |
| **A-UCS** | 辅助 | A-UCS 配置加载与 RCU 热重载的 KUnit 测试；`airy_defconfig` 锁定五大选型的回归测试 |
| **A-ULS** | **核心** | sched_tac 调度测试——验证 Agent 8 态生命周期与 Linux 进程状态的映射；纯 C LSM 钩子注册与 capability 缓存测试；Micro-Supervisor 冷酷执法 + Macro-Supervisor 温情裁决的双层测试 |
| **A-IPC** | **核心** | IPC 零拷贝测试——验证 `IORING_OP_URING_CMD` fastpath 性能（~160ns SLA）、Ring 生命周期解耦、离线缓存校验、Reconciliation 三原则 |

### 4.1 Unify Design 权威源引用

- A-UEF 总纲：[../10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md) §4
- A-ULP 总纲：[../10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md) §5
- A-ULS 总纲：[../10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md) §7
- A-IPC 总纲：[../10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md) §8
- [SC] 契约校验：[../30-interfaces/08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md) + [../30-interfaces/09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md)

---

## 5. 相关文档

- [AirymaxOS 总览](../README.md)：v1.0 技术选型声明 + 20 子目录索引
- [架构设计层](../10-architecture/README.md)：Unify Design 总纲 + IRON-9 v3 四层模型
- [10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md)：Airymax Unify Design 总纲（SSoT）
- [10-architecture/06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md)：IRON-9 v3 四层模型（SSoT）
- [20-modules/08-tests-linux.md](../20-modules/08-tests-linux.md)：tests-linux 子仓设计
- [30-interfaces/08-sc-error-contract.md](../30-interfaces/08-sc-error-contract.md)：A-UEF [SC] error.h 契约
- [30-interfaces/09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md)：A-ULP [SC] log_types.h 契约
- [30-interfaces/10-sc-sched-extension.md](../30-interfaces/10-sc-sched-extension.md)：sched_tac [SC] sched.h 契约
- [50-engineering-standards/09-ssot-registry.md](../50-engineering-standards/09-ssot-registry.md)：SSoT v2 单一权威源注册表
- [70-build-system/03-ci-cd-pipeline.md](../70-build-system/03-ci-cd-pipeline.md)：CI/CD 流水线（`sc-dual-ci.yml` + `ssot-validate.yml`）
- [110-security/README.md](../110-security/README.md)：安全测试（纯 C LSM + capability）
- [170-performance/README.md](../170-performance/README.md)：性能工程（IPC fastpath SLA）

---

## 6. 参考材料

- Linux 6.6 `lib/kunit/`（KUnit 框架）
- Linux 6.6 `tools/testing/selftests/`（kselftest，含 `sched/` 子目录）
- Linux 6.6 `Documentation/dev-tools/testing-overview.rst`（测试工具全景）
- Linux 6.6 `lib/test_*.c`（内核自检样本）
- seL4 项目（形式化验证参考，capDL / l4v 工具链）

---

## 7. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.1.1 | 2026-07-13 | 初始版本，README + 01 + 02 文档奠基，确立 KUnit/kselftest 核心机制 |
| v1.0 | 2026-07-17 | 升级为 v1.0：新增sched_tac / IORING_OP_URING_CMD / 纯 C LSM / alloc_pages + mmap / IRON-9 v3 四层模型五大技术选型声明（测试体系作为验证守护者）；新增 Airymax Unify Design 映射（sched_tac 调度测试 + IPC 零拷贝测试 + [SC] 逐字节校验为核心）；文档索引对齐实际目录文件 |

---

## 8. 回归测试框架（v1.1 增量补强）

> **补强背景**：80-testing/ 现有 10 卷文档覆盖单元测试、系统测试、动态分析、覆盖率等正向验证，但缺少专门的回归测试框架定义。v1.1 引入 Capability Folding 架构后，fastpath C-S9 内联校验、`agent_caps[]` 静态数组等改动一旦引入回归，将影响全部 120 个 Agent syscall。本章节定义回归测试套件组织、触发机制、git bisect 自动化与通过率门槛。

### 8.1 回归测试套件组织

```
tools/testing/airy_regression/
├── README.md                          # 回归测试使用说明
├── airy_regression_runner.sh          # 回归测试主执行器
├── airy_regression_baseline.json      # 回归基线（已知通过的用例集）
├── airy_regression_report.py          # 报告生成器
├── suites/
│   ├── kunit_regression.list          # KUnit 回归用例列表
│   ├── kselftest_regression.list      # kselftest 回归用例列表
│   ├── contract_regression.list       # Agent 契约回归用例（17 条契约）
│   ├── cap_folding_regression.list    # v1.1 Capability Folding 回归用例
│   ├── perf_regression.list           # 性能回归基准（fastpath/io_uring/Badge）
│   └── daemon_recovery_regression.list # 多 daemon 灾难恢复回归用例
├── baselines/
│   ├── v1.1.0_baseline.json           # v1.1.0 发布时的基线快照
│   └── v1.0.1_baseline.json           # v1.0.1 发布时的基线快照
└── scripts/
    ├── airy_bisect.sh                 # git bisect 自动化脚本
    └── airy_regression_diff.py        # PR 影响范围分析
```

### 8.2 触发机制

| 触发时机 | 范围 | 通过率门槛 | 阻断级别 |
|---------|------|-----------|---------|
| 每次 PR merge 触发 | 增量回归（仅 PR 修改模块相关用例） | 100% | 阻断 merge |
| nightly 全量回归 | 全量回归（全部用例） | ≥ 99% | 阻断 release |
| release 前 | 全量回归 + 性能回归 + 长时稳定性 | 100% | 阻断 release |
| hotfix 后 | 增量回归 + 受影响模块全量 | 100% | 阻断 hotfix 合入 |

```yaml
# .github/workflows/regression.yml
name: airy-regression
on:
  push:
    branches: [main]
  pull_request:
    types: [closed]
  schedule:
    - cron: "0 2 * * *"  # UTC 02:00（北京 10:00）nightly 全量
  workflow_dispatch: {}

jobs:
  incremental-regression:
    if: github.event_name == 'pull_request' && github.event.pull_request.merged == true
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - name: Analyze PR impact scope
        run: |
          python3 tools/testing/airy_regression/scripts/airy_regression_diff.py \
            --pr ${{ github.event.pull_request.number }} \
            --output impacted_modules.txt
      - name: Run incremental regression
        run: |
          bash tools/testing/airy_regression/airy_regression_runner.sh \
            --mode incremental \
            --impacted impacted_modules.txt \
            --baseline airy_regression_baseline.json

  nightly-full-regression:
    if: github.event_name == 'schedule'
    runs-on: ubuntu-24.04-large
    timeout-minutes: 240
    steps:
      - uses: actions/checkout@v4
      - name: Run full regression
        run: |
          bash tools/testing/airy_regression/airy_regression_runner.sh \
            --mode full \
            --baseline airy_regression_baseline.json \
            --threshold 99
      - name: Generate report
        run: |
          python3 tools/testing/airy_regression/airy_regression_report.py \
            --input results.json \
            --output report.html
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: regression-report
          path: report.html
```

### 8.3 git bisect 自动化

`scripts/airy_bisect.sh` 自动定位引入回归的 commit：

```bash
#!/bin/bash
# scripts/airy_bisect.sh
# 用法: airy_bisect.sh <bad_commit> <good_commit> <test_command>
# 示例: airy_bisect.sh HEAD v1.0.1 "airy_regression_runner.sh --suite cap_folding"

set -euo pipefail

BAD_COMMIT="${1:?Usage: airy_bisect.sh <bad> <good> <test_cmd>}"
GOOD_COMMIT="${2:?}"
TEST_CMD="${3:?}"

echo "[airy_bisect] Locating regression: bad=$BAD_COMMIT good=$GOOD_COMMIT"

git bisect start
git bisect bad "$BAD_COMMIT"
git bisect good "$GOOD_COMMIT"

# 自动 bisect：运行测试命令，退出码 0=good，非 0=bad
git bisect run bash -c "$TEST_CMD"

# 显示结果
BISECT_RESULT=$(git bisect view --stat)
echo "[airy_bisect] Regression introduced by:"
echo "$BISECT_RESULT"

git bisect reset

# 自动创建 issue
if [ -n "$GITHUB_TOKEN" ]; then
    COMMIT=$(echo "$BISECT_RESULT" | head -1 | awk '{print $1}')
    gh issue create \
        --title "Regression introduced by $COMMIT" \
        --body "Bisect result: $BISECT_RESULT" \
        --label regression,critical
fi
```

### 8.4 回归基线维护

`airy_regression_baseline.json` 记录已知通过的用例集，作为回归对比基线：

```json
{
  "version": "v1.1.0",
  "baseline_date": "2026-07-15",
  "total_suites": 6,
  "total_cases": 1247,
  "suites": [
    {
      "name": "kunit_regression",
      "cases": 312,
      "expected_pass": 312,
      "duration_sla_s": 30
    },
    {
      "name": "kselftest_regression",
      "cases": 187,
      "expected_pass": 187,
      "duration_sla_s": 120
    },
    {
      "name": "contract_regression",
      "cases": 17,
      "expected_pass": 17,
      "duration_sla_s": 10
    },
    {
      "name": "cap_folding_regression",
      "cases": 48,
      "expected_pass": 48,
      "duration_sla_s": 60
    },
    {
      "name": "perf_regression",
      "cases": 5,
      "expected_pass": 5,
      "duration_sla_s": 1800
    },
    {
      "name": "daemon_recovery_regression",
      "cases": 678,
      "expected_pass": 678,
      "duration_sla_s": 300
    }
  ],
  "thresholds": {
    "pr_merge_pass_rate_pct": 100,
    "nightly_pass_rate_pct": 99,
    "release_pass_rate_pct": 100
  }
}
```

### 8.5 通过率门槛

**OS-TEST-116**：回归测试通过率门槛——每次 PR merge 触发的增量回归必须 100% 通过（任一失败阻断 merge）；nightly 全量回归通过率 ≥ 99%（< 99% 阻断 release）；release 前全量回归必须 100% 通过。

**OS-STD-093**：回归测试失败的用例必须在 24 小时内修复或回滚；连续 2 次 nightly 回归失败必须暂停 PR 合入（仅 critical fix 例外），直至回归恢复 ≥ 99%。

**OS-KER-172**：回归基线 `airy_regression_baseline.json` 必须随每次 release 更新快照（`baselines/v{version}_baseline.json`）；任一 release 后基线未更新即视为 release 流程缺陷。

---

## 9. 长时稳定性测试（v1.1 增量补强）

> **补强背景**：80-testing/ 现有测试均为短时（分钟级至小时级），缺少 24h+ 高负载稳定性测试。v1.1 Capability Folding 架构的 `agent_caps[1024]` 静态数组、Epoch 单调递增计数器等数据结构在长期高负载下可能累积微小问题（内存碎片、计数器回绕、RCU grace period 延迟累积）。本章节定义 24h nightly + 7d release 前的长时稳定性测试。

### 9.1 测试场景

| 场景 | 负载 | 持续时间 | 触发时机 |
|------|------|---------|---------|
| LSO-1 | 100 Agent 并发 IPC（每秒 10 万次 io_uring 提交） | 24h | nightly |
| LSO-2 | 持续 Badge 编译/撤销（每秒 1 万次） | 24h | nightly |
| LSO-3 | Agent 创建/销毁循环（每秒 100 次） | 24h | nightly |
| LSO-4 | 混合负载（LSO-1 + LSO-2 + LSO-3） | 7d | release 前 |

### 9.2 持续时间

- **nightly**：24 小时（UTC 02:00 启动，次日 UTC 02:00 结束）
- **release 前**：7 天（release 候选版本构建后立即启动，7×24 小时持续）

### 9.3 监控指标

| 指标 | 采集方法 | 告警阈值 | 失败阈值 |
|------|---------|---------|---------|
| 内存泄漏 | kmemleak 每 10 分钟扫描 | 单次扫描 ≥ 1 个泄漏 | 累计 ≥ 10 个泄漏 |
| 内存增长 | `/proc/meminfo` + `airy_agent_mem_stats` | 24h 增长 > 3% | 24h 增长 > 5% |
| CPU 调度延迟 | `perf sched` + `/proc/sched_debug` | P99 > 5ms | P99 > 10ms |
| Ring 队列深度 | `airy_ipc_ring_depth_stats()` | 平均深度 > 80% 容量 | 队列满 ≥ 1 次 |
| Epoch 计数器 | `airy_epoch_current()` | 24h 内增长异常 | Epoch 回退（致命） |
| `agent_caps[]` 利用率 | `airy_cap_usage_pct()` | > 90% | 100%（分配失败） |
| daemon CPU 占用 | `systemd-cgtop` | 单 daemon > 80% 持续 5min | 单 daemon > 95% 持续 1min |

### 9.4 通过标准

- **0 内核 oops / panic**：任一 oops/panic 即失败
- **0 daemon 崩溃**：12 daemon 任一崩溃即失败（systemd 自动重启不计为崩溃，但记录）
- **内存增长 < 5%**：24h 后总内存占用增长 < 5%
- **0 KASAN/KCSAN/lockdep 报告**：动态分析全程无报告
- **0 契约违反**：`airy_agent_contract_violation` tracepoint 全程无触发
- **性能不退化**：24h 结束时的 fastpath 延迟 P99 与开始时差值 < 5%

### 9.5 工具

```bash
# stress-ng 提供基础负载
stress-ng --cpu 4 --io 2 --vm 2 --vm-bytes 4G --timeout 24h &

# airy-stress 自定义工具提供 Agent 专属负载
airy-stress \
  --agents 100 \
  --ipc-rate 100000 \
  --badge-rate 10000 \
  --spawn-stop-rate 100 \
  --duration 24h \
  --monitor kmemleak,mem_growth,sched_latency,ring_depth \
  --report airy_stress_24h_report.json

# 7d release 前测试
airy-stress \
  --agents 100 \
  --ipc-rate 100000 \
  --badge-rate 10000 \
  --spawn-stop-rate 100 \
  --duration 7d \
  --mode mixed \
  --monitor all \
  --report airy_stress_7d_report.json
```

**OS-TEST-117**：nightly 长时稳定性测试必须运行 24 小时，覆盖 LSO-1/LSO-2/LSO-3 三类场景；任一失败标准触发即标记 nightly 失败。

**OS-KER-173**：release 前必须运行 7 天长时稳定性测试（LSO-4 混合负载）；7 天内任一内核 oops、daemon 崩溃、内存增长 > 5% 即阻断 release。

---

## 10. Live Patch 兼容性测试（v1.1 增量补强）

> **补强背景**：80-testing/ 未定义 live patch（kpatch/kexec）对 AirymaxOS 数据结构的影响。v1.1 Capability Folding 架构的 `agent_caps[1024]` 静态数组布局、Epoch 全局计数器、Badge 64-bit 格式等若在 live patch 中被修改，将导致运行中 Agent 的 Badge 全部失效或内存损坏。本章节定义 live patch 兼容性测试范围与回滚机制。

### 10.1 测试范围

| 测试类型 | 工具 | 覆盖范围 |
|---------|------|---------|
| kpatch 热补丁 | kpatch + livepatch | sec_d / cogn_d / mem_d / gateway_d daemon 修复 |
| kexec 重启 | kexec + kdump | 内核镜像替换（warm reboot） |
| config 迁移 | `airy_migrate_config` | `airy_defconfig` 配置项变更 |

### 10.2 数据结构兼容性

| 数据结构 | 兼容性约束 | 验证方法 |
|---------|-----------|---------|
| `agent_caps[1024]` 布局 | 数组大小、元素结构体布局不变 | `airy_cap_layout_hash()` live patch 前后比对 |
| Epoch 全局计数器 | 单调递增，live patch 后不得回退 | `airy_epoch_current()` live patch 后 ≥ patch 前 |
| Badge 64-bit 格式 | `Epoch<<48 \| RandomTag<<16 \| Perms` 字段位移不变 | `airy_cap_badge_format_check()` 验证字段掩码 |
| Agent 状态机 | 8 态 14 合法转换不变 | `airy_agent_transition_legal_table` 哈希比对 |
| Token 预算结构 | `airy_token_budget` 结构体布局不变 | `sizeof(struct airy_token_budget)` 比对 |

### 10.3 测试用例

```c
/* tools/testing/selftests/airy_livepatch/cap_compat.c */
#include "../kselftest_harness.h"
#include <sys/airy_cap.h>
#include <sys/airy_livepatch.h>

FIXTURE(airy_livepatch_compat) {
    u64 layout_hash_before;
    u64 epoch_before;
};

FIXTURE_SETUP(airy_livepatch_compat) {
    self->layout_hash_before = airy_cap_layout_hash();
    self->epoch_before = airy_epoch_current();
}

TEST_F(airy_livepatch_compat, agent_caps_layout_unchanged) {
    /* 应用 live patch（修复 sec_d 中的某个 bug） */
    ASSERT_EQ(0, system("kpatch load sec_d_fix.ko"));

    /* 验证 agent_caps[] 布局哈希不变 */
    u64 hash_after = airy_cap_layout_hash();
    ASSERT_EQ(self->layout_hash_before, hash_after);
}

TEST_F(airy_livepatch_compat, epoch_monotonic_after_patch) {
    ASSERT_EQ(0, system("kpatch load sec_d_fix.ko"));

    /* 验证 Epoch 单调递增（live patch 后 ≥ patch 前） */
    u64 epoch_after = airy_epoch_current();
    ASSERT_GE(epoch_after, self->epoch_before);
}

TEST_F(airy_livepatch_compat, existing_badges_still_valid) {
    /* live patch 前编译一个 Badge */
    u64 badge = airy_cap_badge_compile(42, AIRY_CAP_PERMS_DEFAULT);
    ASSERT_NE(0, badge);

    /* 应用 live patch */
    ASSERT_EQ(0, system("kpatch load sec_d_fix.ko"));

    /* 验证 live patch 后该 Badge 仍然有效 */
    ASSERT_EQ(true, airy_cap_badge_ok(42, badge));
}

TEST_F(airy_livepatch_compat, cap_integrity_after_patch) {
    /* 应用 live patch */
    ASSERT_EQ(0, system("kpatch load sec_d_fix.ko"));

    /* 验证 agent_caps[] 完整性 */
    ASSERT_EQ(0, airy_cap_integrity_check());
}
```

### 10.4 失败回滚

kpatch 回滚机制：

```bash
#!/bin/bash
# scripts/airy_livepatch_rollback.sh
# 用法: airy_livepatch_rollback.sh <patch_module>

PATCH="${1:?Usage: airy_livepatch_rollback.sh <patch_module>}"

echo "[airy_livepatch] Pre-rollback integrity check..."
if ! airy_cap_integrity_check; then
    echo "[airy_livepatch] INTEGRITY FAILED, rolling back..."

    # 回滚 live patch
    kpatch unload "$PATCH"

    # 验证回滚后完整性
    sleep 2
    if airy_cap_integrity_check; then
        echo "[airy_livepatch] Rollback SUCCESS, integrity restored"
        # 触发告警
        gh issue create --title "Live patch $PATCH rolled back" \
            --label livepatch-failure,critical
        exit 0
    else
        echo "[airy_livepatch] Rollback FAILED, integrity still broken"
        # 触发紧急回滚至上一稳定版本
        systemctl airy-emergency-rollback
        exit 1
    fi
else
    echo "[airy_livepatch] Integrity OK, no rollback needed"
    exit 0
fi
```

**OS-TEST-118**：Live patch 兼容性测试必须覆盖 kpatch + kexec + config 迁移三类；任一数据结构兼容性检查失败即标记测试失败，自动触发回滚。

**OS-KER-174**：Live patch 修改 `agent_caps[]` 布局或 Badge 64-bit 格式即视为不兼容变更，禁止通过 live patch 部署，必须走完整 release 流程（含升级/降级测试，见 §11）。

---

## 11. 升级/降级测试（v1.1 增量补强）

> **补强背景**：80-testing/ 未定义 AirymaxOS 版本升级/降级测试。v1.1 Capability Folding 架构引入 `agent_caps[1024]` 静态数组与 Badge 64-bit 格式，与 v1.0.x 的 41 ID 权限模型不兼容，必须定义明确的升级/降级路径与数据迁移。本章节定义版本间升级、降级、config 迁移、ABI 兼容性测试。

### 11.1 升级路径

| 升级路径 | 兼容性 | 数据迁移 | 测试范围 |
|---------|--------|---------|---------|
| 0.1.1 → 1.0.1 | ABI 不兼容 | `airy_migrate_config` 自动迁移 | config 迁移 + Agent 状态恢复 |
| 1.0.1 → 1.1.0 | ABI 不兼容（41 ID → Capability Folding） | `airy_migrate_caps` 迁移权限模型 | config 迁移 + `agent_caps[]` 重建 + Agent 重新认证 |
| 0.1.1 → 1.1.0 | ABI 不兼容 | 两步迁移（0.1.1 → 1.0.1 → 1.1.0） | 全量升级测试 |

### 11.2 降级路径

| 降级路径 | 适用场景 | 数据兼容性 | 测试范围 |
|---------|---------|-----------|---------|
| 1.1.0 → 1.0.1 | 紧急回滚（v1.1.0 发现 critical bug） | `agent_caps[]` 数据丢失（降级后重建 41 ID 模型） | 紧急回滚 + Agent 重新认证 |
| 1.0.1 → 0.1.1 | 不支持 | — | 禁止降级 |

### 11.3 测试范围

| 测试项 | 升级 | 降级 |
|--------|------|------|
| config 迁移 | `airy_migrate_config` 自动迁移 `airy_defconfig` | config 回滚至上一版本 |
| `agent_caps[]` 兼容性 | v1.0.x 41 ID → v1.1 Badge 迁移（`airy_migrate_caps`） | v1.1 Badge 丢弃，重建 v1.0.x 41 ID |
| ABI 兼容性 | 用户态 daemon 重新编译（v1.1 [SC] 头文件） | daemon 回滚至 v1.0.x 二进制 |
| Agent 状态恢复 | 升级中 RUNNING Agent 暂停，升级后恢复 | 降级中 RUNNING Agent 强制 STOPPED |
| 审计哈希链 | 升级前后哈希链连续 | 降级后哈希链标注"版本切换"断点 |

### 11.4 数据迁移工具

```bash
# airy_migrate_config: config 迁移
airy_migrate_config \
    --from-version 1.0.1 \
    --to-version 1.1.0 \
    --config /etc/airy/airy_defconfig \
    --backup /etc/airy/airy_defconfig.bak.1.0.1

# airy_migrate_caps: 权限模型迁移（41 ID → Capability Folding）
airy_migrate_caps \
    --from-model id41 \
    --to-model cap_folding \
    --source /var/lib/airy/caps_41.json \
    --target /var/lib/airy/agent_caps.bin \
    --backup /var/lib/airy/caps_41.json.bak

# 输出示例
# [airy_migrate_caps] Migrating 1024 agents from id41 to cap_folding...
# [airy_migrate_caps] Agent 0: id41 perms=0x1F → Badge=0x000100020000001F
# [airy_migrate_caps] Agent 1: id41 perms=0x0F → Badge=0x000100030000000F
# ...
# [airy_migrate_caps] Migration complete: 1024 agents migrated, 0 errors
```

### 11.5 测试用例

```c
/* tools/testing/selftests/airy_upgrade/cap_migration.c */
#include "../kselftest_harness.h"
#include <sys/airy_migrate.h>

TEST(airy_upgrade, migrate_41_id_to_cap_folding) {
    /* 准备 v1.0.x 41 ID 权限模型数据 */
    ASSERT_EQ(0, system("airy_setup_41_id_model --agents 1024"));

    /* 执行迁移 */
    ASSERT_EQ(0, system("airy_migrate_caps --from-model id41 "
                         "--to-model cap_folding"));

    /* 验证：所有 1024 Agent 均有合法 Badge */
    ASSERT_EQ(1024, airy_cap_count_valid_badges());

    /* 验证：权限语义保留（41 ID 的 0x1F → Badge 的 Perms=0x1F） */
    for (int i = 0; i < 1024; i++) {
        u64 badge = airy_cap_get_badge(i);
        u32 perms = badge & 0xFFFF;
        u32 expected = airy_41_id_get_perms(i);
        ASSERT_EQ(expected, perms);
    }
}

TEST(airy_upgrade, rollback_1_1_0_to_1_0_1) {
    /* 模拟 v1.1.0 紧急回滚至 v1.0.1 */
    ASSERT_EQ(0, system("airy_emergency_rollback --to 1.0.1"));

    /* 验证：系统恢复至 v1.0.1 */
    ASSERT_EQ("1.0.1", airy_kernel_version());

    /* 验证：Agent 数据未丢失（agent_caps[] 重建为 41 ID） */
    ASSERT_EQ(1024, airy_41_id_count_valid());

    /* 验证：审计哈希链标注版本切换断点 */
    ASSERT_EQ(1, airy_audit_count_version_switch_markers());
}

TEST(airy_upgrade, zero_data_loss) {
    /* 升级前记录所有 Agent 状态 */
    char snapshot_before[65536];
    ASSERT_GT(0, airy_agent_snapshot_all(snapshot_before, sizeof(snapshot_before)));

    /* 执行升级 1.0.1 → 1.1.0 */
    ASSERT_EQ(0, system("airy_upgrade --from 1.0.1 --to 1.1.0"));

    /* 升级后记录所有 Agent 状态 */
    char snapshot_after[65536];
    ASSERT_GT(0, airy_agent_snapshot_all(snapshot_after, sizeof(snapshot_after)));

    /* 验证：Agent 状态语义保留（允许格式不同，但语义一致） */
    ASSERT_EQ(0, airy_agent_snapshot_semantic_compare(
        snapshot_before, snapshot_after));
}
```

### 11.6 通过标准

**OS-TEST-119**：升级测试必须覆盖 0.1.1 → 1.0.1 → 1.1.0 完整升级路径；升级后所有 Agent 正常运行（无状态丢失、无权限错误）即通过。

**OS-TEST-120**：降级测试必须覆盖 1.1.0 → 1.0.1 紧急回滚路径；降级后 0 数据丢失（Agent 状态、Token 账本、L1-L4 记忆配额均完整）即通过。

**OS-KER-175**：升级/降级测试失败的版本禁止 release；`airy_migrate_caps` 迁移失败的 Agent 必须在迁移报告中显式列出，由人工介入处理。

---

## 12. 混沌工程（v1.1 增量补强）

> **补强背景**：80-testing/ 现有测试均为确定性故障注入（如 §15 多 daemon 故障注入），缺少随机故障注入的混沌工程测试。v1.1 的 12 daemon 协同架构在随机故障组合下可能暴露未预见的级联失败。本章节定义随机故障注入的混沌工程测试框架。

### 12.1 故障注入类型

| 故障类型 | 注入机制 | 持续时间 | 影响范围 |
|---------|---------|---------|---------|
| daemon 杀死 | `systemctl kill --signal=SIGKILL <daemon>` | 随机 1-60s | 单 daemon 不可用 |
| 网络分区 | `iptables` + `tc netem` 模拟分区 | 随机 10-300s | gateway_d 与其他 daemon 通信中断 |
| 磁盘满 | `dd if=/dev/zero of=/var/lib/airy/fill bs=1M` | 随机 10-120s | mem_d 持久化失败、logger_d 日志写入失败 |
| CPU 满载 | `stress-ng --cpu $(nproc) --timeout <s>` | 随机 10-60s | 全系统调度延迟激增 |
| 内存压力 | `stress-ng --vm 4 --vm-bytes 4G` | 随机 10-120s | OOM 风险、kmemleak 误报 |

### 12.2 工具

- **systemd-cgred**：通过 cgroup 限制 daemon 资源
- **stress-ng**：提供 CPU/内存/IO 压力负载
- **airy-chaos**：agentrt-linux 专属混沌引擎，随机选择故障组合并注入

```bash
# airy-chaos 工具用法
airy-chaos \
  --fault-types daemon_kill,network_partition,disk_full,cpu_stress,mem_pressure \
  --concurrent-faults 1-3 \
  --duration 30m \
  --verify self_heal,data_consistency,audit_integrity \
  --report airy_chaos_report.json
```

### 12.3 测试矩阵

每次混沌测试随机选择 1-3 个故障同时注入，覆盖以下组合：

| 组合编号 | 故障组合 | 预期行为 |
|---------|---------|---------|
| C1 | daemon_kill(sec_d) | sec_d 自愈，agent_caps[] 完整 |
| C2 | daemon_kill(sec_d) + network_partition | sec_d 自愈 + gateway_d 通信恢复 |
| C3 | disk_full + daemon_kill(logger_d) | logger_d 自愈 + 磁盘清理后日志恢复 |
| C4 | cpu_stress + mem_pressure + daemon_kill(cogn_d) | cogn_d 自愈 + 系统降级运行 |
| C5 | 随机 3 个故障组合 | 系统不得崩溃，自愈后数据一致 |

### 12.4 持续时间与频率

- **单次测试**：30 分钟（含故障注入 5min + 自愈观察 10min + 数据一致性验证 15min）
- **执行频率**：每天 1 次（由 `airy-chaos-engine.service` 定时执行）

### 12.5 验证

| 验证项 | 验证方法 | 通过标准 |
|--------|---------|---------|
| 系统自愈 | 所有 daemon 在故障消除后 30s 内恢复 | 12 daemon 全部 active |
| 数据一致性 | `airy_cap_integrity_check` + `airy_audit_hash_chain_verify` | 0 项损坏 |
| 审计完整性 | `airy_audit_hash_chain_verify` | 哈希链完整，无断裂 |
| Agent 状态 | `airy_agent_state_machine_consistent` | 0 个 Agent 处于非法状态 |
| 性能恢复 | 故障消除后 5min 内 fastpath 延迟恢复至基线 ±5% | P99 ≤ 10.5ns |

### 12.6 自动化

```ini
# /etc/systemd/system/airy-chaos-engine.service
[Unit]
Description=AirymaxOS Chaos Engineering Engine
After=network.target airy-daemons.target

[Service]
Type=oneshot
ExecStart=/usr/bin/airy-chaos \
  --fault-types daemon_kill,network_partition,disk_full,cpu_stress,mem_pressure \
  --concurrent-faults 1-3 \
  --duration 30m \
  --verify self_heal,data_consistency,audit_integrity \
  --report /var/log/airy/chaos/airy_chaos_$(date +%Y%m%d).json
ExecStartPost=/bin/bash -c 'if [ $? -ne 0 ]; then systemctl start airy-chaos-alert; fi'

# 每日定时执行
[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true

[Install]
WantedBy=multi-user.target
```

```bash
# 启用混沌引擎
systemctl enable --now airy-chaos-engine.service

# 查看历史报告
ls /var/log/airy/chaos/
# airy_chaos_20260718.json  airy_chaos_20260719.json  ...

# 解析报告
airy-chaos-report --input /var/log/airy/chaos/airy_chaos_20260718.json
```

**OS-TEST-121**：混沌工程测试必须每天执行 1 次，每次 30 分钟，随机注入 1-3 个故障；任一验证项失败（自愈失败、数据损坏、审计断裂）即标记当日混沌失败，自动创建 issue。

**OS-KER-176**：混沌工程测试发现的级联失败（如 sec_d 故障导致 cogn_d 连锁崩溃）必须进行根因分析并修复；连续 3 次混沌测试失败的故障组合必须暂停 release，直至该组合稳定性恢复。

---

> **文档结束** | agentrt-linux 测试体系设计 v1.0 | 维护者：开源极境工程与规范委员会 | "From data intelligence emerges."
