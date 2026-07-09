Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# Airymax Documentation Center

**Language:** English | [简体中文](README_zh.md)

**Latest**: 2026-06-09
**Status**: Maintained
**Path**: OpenAirymax/Docs/README.md
---

## 📚 Documentation Navigation

Welcome to the Airymax Agent Operating System documentation center. This documentation is organized in a layered structure, from beginner to advanced, to meet the needs of different levels.

### 🎯 Quick Entry

| Role | Recommended Reading Path | Estimated Time |
|------|--------------------------|----------------|
| **Beginner** | Quick Start → Installation Guide → Configuration Guide | 30 minutes |
| **Developer** | API Reference → Coding Standards → Create Agent/Skill | 2 hours |
| **Ops Engineer** | Deployment Guide → Monitoring & Ops → Troubleshooting | 1.5 hours |
| **Architect** | Design Principles → MicroCore Architecture → CoreLoopThree | 3 hours |

---

## 📖 Documentation Categories

### 1️⃣ Basic Theories

The foundational design pillars of Airymax and required reading for understanding the design philosophy:

- [**MCIS (Multi-body Control Intelligence System)**](00-basic-theories/01-mcis.md) — Multi-body Control Intelligence System ([中文](00-basic-theories/01-mcis-cn.md))
- [**Cognition Layer Design**](00-basic-theories/02-cognition-design.md) — Dual thinking system and three-layer architecture ([中文](00-basic-theories/02-cognition-design-cn.md))
- [**Memory Layer Design**](00-basic-theories/03-memory-design.md) — Four-layer memory rovol model ([中文](00-basic-theories/03-memory-design-cn.md))
- [**Design Principles**](00-basic-theories/04-design-principles.md) — Introduction to the five-dimensional orthogonal design principles ([中文](00-basic-theories/04-design-principles-cn.md))
- [**Design Philosophy**](10-architecture/40-philosophy/01-design-philosophy.md) — Detailed exposition of MCIS and the five-dimensional orthogonal design system
- [**Architectural Principles (Complete)**](00-architectural-principles.md) — Detailed exposition of the 24 S/K/C/E/A principles

---

### 2️⃣ Guides

Guided tutorials for new users to get started quickly:

- [**Quick Start**](60-guides/01-getting-started.md) — From zero to Hello World in 5 minutes
- [**Configuration Guide**](60-guides/04-configuration-guide.md) — Complete description of configuration options
- [**Deployment Guide**](60-guides/03-deployment-guide.md) — Production deployment best practices (Docker/K8s/Monitoring/Security)
- [**Create Agent**](60-guides/24-create-agent.md) — Complete Agent development workflow
- [**Create Skill**](60-guides/25-create-skill.md) — Complete Skill development workflow
- [**Plugin SDK Tutorial**](60-guides/17-plugin-sdk-tutorial.md) — Complete Plugin SDK development guide
- [**Prompt Engineering Guide**](60-guides/19-prompt-engineering.md) — Full lifecycle of prompt templates/injection/tuning
- [**Migration Guide**](60-guides/10-migration-guide.md) — Version upgrades and data migration
- [**Build Guide**](60-guides/02-build-guide.md) — Build system and compilation options
- [**Testing Standards**](60-guides/15-testing-standards.md) — Test layering and coverage standards
- [**Performance Tuning**](60-guides/09-performance-tuning.md) — Kernel parameters and performance optimization
- [**Protocol Integration**](60-guides/18-protocol-integration.md) — Multi-protocol adaptation and routing

---

### 3️⃣ Architecture

In-depth understanding of Airymax's design philosophy and technical implementation:

- [**MicroCore Architecture**](10-architecture/04-microcorert.md) — Implementation of K-1~K-4 principles
- [**CoreLoopThree**](10-architecture/02-coreloopthree.md) — Cognition→Execution→Memory
- [**Memory Rovol System**](10-architecture/03-memoryrovol.md) — L1→L2→L3→L4 four-layer memory
- [**IPC Communication**](10-architecture/30-kernel/01-ipc.md) — Binder/Channel/Buffer inter-process communication
- [**Syscall Architecture**](10-architecture/05-syscall.md) — Unified interface between user space and kernel space
- [**Logging System Architecture**](10-architecture/50-services/02-logging.md) — Cross-language observability and dynamic feedback regulation
- [**Architecture Overview**](10-architecture/01-system-architecture.md) — System overview and layered architecture diagram
- [**C Language Boundary**](10-architecture/30-kernel/02-c-language-boundary.md) — C core responsibility scope and FFI boundary

---

### 4️⃣ API Reference

Complete interface documentation, including request examples and response formats:

**Syscall API**:

- [**API Overview**](30-api/README.md) — API hierarchy and design philosophy
- [**Gateway API Reference**](30-api/02-api-reference.md) — HTTP/WebSocket endpoint API reference
- [**CLI Command Reference**](30-api/03-cli-reference.md) — Complete commands for the unified CLI tool
- [**Task Management API**](30-api/60-syscalls/05-task.md) — submit/query/wait/cancel full task lifecycle
- [**Memory Management API**](30-api/60-syscalls/02-memory.md) — write/query/evolve/forget four-layer memory operations
- [**Session Management API**](30-api/60-syscalls/03-session.md) — create/get/close/list session management
- [**Observability API**](30-api/60-syscalls/06-telemetry.md) — metrics/traces telemetry interface
- [**Agent Management API**](30-api/60-syscalls/01-agent.md) — spawn/terminate/invoke Agent management

**Multi-language SDK**:

- [**Python SDK**](30-api/70-toolkit/20-python/README.md) — Python language binding API
- [**Go SDK**](30-api/70-toolkit/10-go/README.md) — Go language binding API
- [**Rust SDK**](30-api/70-toolkit/30-rust/README.md) — Rust language binding API
- [**TypeScript SDK**](30-api/70-toolkit/40-typescript/README.md) — TypeScript language binding API

**Core Algorithms**:

- [**Algorithm Implementation Docs**](30-api/10-algorithms/README.md) — Document processing/search index/quality validation/performance optimization core algorithms

---

### 5️⃣ Development

Essential knowledge for contributing to Airymax development:

**Contribution & Testing**:

- [**Contribution Guide**](../agentrt/CONTRIBUTING.md) — Complete PR submission workflow
- [**Testing Guide**](60-guides/14-testing-guide.md) — Unit tests, integration tests, E2E tests

**Coding Standards**:

- [**C Coding Style Standard**](50-specifications/30-coding-standard/01-c-coding-style.md) — C naming/functions/error handling/concurrency/security standards
- [**C++ Coding Style Standard**](50-specifications/30-coding-standard/05-cpp-coding-style.md) — C++ coding standards
- [**Python Coding Style Standard**](50-specifications/30-coding-standard/12-python-coding-style.md) — Python type design/async/error handling standards
- [**JavaScript Coding Style Standard**](50-specifications/30-coding-standard/08-javascript-coding-style.md) — JavaScript/TypeScript coding standards
- [**Naming Conventions**](50-specifications/30-coding-standard/11-naming-conventions.md) — Component/file/function/type/constant naming conventions
- [**Code Comment Template**](50-specifications/30-coding-standard/03-code-comment-template.md) — Doxygen/docstring comment standards
- [**Logging Standard**](50-specifications/30-coding-standard/10-log-standard.md) — Log level/content/format/quality standards

**Secure Coding**:

- [**Security Design Standard**](50-specifications/30-coding-standard/15-security-design.md) — D1~D4 four-layer protection/encryption/authentication/privacy protection
- [**C/C++ Secure Coding Standard**](50-specifications/30-coding-standard/02-c-cpp-secure-coding.md) — C/C++ secure coding practices
- [**Java Secure Coding Standard**](50-specifications/30-coding-standard/09-java-secure-coding.md) — Java secure coding practices

---

### 6️⃣ Operations

Operational assurance for production environments:

- [**Docker Deployment**](../Docker/README.md) — Complete containerized deployment solution
- [**Monitoring & Ops**](60-guides/07-monitoring-guide.md) — Prometheus+Grafana monitoring stack
- [**Backup & Recovery**](60-guides/11-backup-recovery.md) — Data backup and disaster recovery
- [**Kernel Tuning**](60-guides/09-performance-tuning.md) — Kernel parameter tuning and performance optimization

---

### 7️⃣ Troubleshooting

Common issues and solutions:

- [**FAQ**](60-guides/23-troubleshooting-faq.md) — Troubleshooting and high-frequency issues
- [**Error Diagnosis**](60-guides/08-diagnosis-guide.md) — Log analysis and problem localization
- [**Known Issues**](60-guides/22-known-issues.md) — Known bugs and temporary workarounds

---

### 8️⃣ Specifications

The standardized specification system for the Airymax project:

**Contract Specifications**:

- [**Contract Overview**](50-specifications/10-contracts/06-glossary-index.md) — Glossary and quick index
- [**Agent Contract**](50-specifications/10-contracts/01-agent-contract.md) — Agent capability description specification ([Schema](50-specifications/10-contracts/01-agent-contract-schema.json))
- [**Skill Contract**](50-specifications/10-contracts/02-skill-contract.md) — Skill capability description specification ([Schema](50-specifications/10-contracts/02-skill-contract-schema.json))
- [**Protocol Specification**](50-specifications/10-contracts/03-protocol-contract.md) — HTTP/WS/Stdio gateway + JSON-RPC 2.0
- [**Syscall API Specification**](50-specifications/10-contracts/04-syscall-api-contract.md) — Syscall interface contract
- [**Logging Format Specification**](50-specifications/10-contracts/05-logging-format.md) — Structured JSON log format

**Integration Standards**:

- [**Integration Standards Overview**](50-specifications/50-integration/README.md) — Index of inter-module integration standards
- [**Manager Configuration Integration Standard**](50-specifications/50-integration/01-integration-standard.md) — Manager module integration with the unified configuration library

**Project Management**:

- [**Error Code Reference**](50-specifications/70-project-erp/02-error-code-reference.md) — Complete error code definitions and handling recommendations
- [**Resource Management Table**](50-specifications/70-project-erp/04-resource-management-table.md) — Resource creation/release/ownership specifications
- [**Software Bill of Materials (SBOM)**](50-specifications/70-project-erp/01-sbom.md) — Component/dependency/license/security information
- [**Module Requirements**](50-specifications/70-project-erp/03-manuals-module-requirements.md) — manuals module requirements and technical specifications

**Terminology**:

- [**Unified Terminology**](50-specifications/10-terminology.md) — Unified Airymax terminology definitions

---

### 9️⃣ White Paper & Templates

- [**Technical White Paper**](90-references/01-white-paper.md) — Official Airymax technical white paper (CN/EN)
- [**Document Template**](90-references/02-template.md) — General document template
- [**API Document Template**](90-references/03-template-api.md) — API document authoring template
- [**Guide Document Template**](90-references/04-template-guide.md) — Guide document authoring template

---

### 🔟 References

Supplementary materials and external links:

- [**Unified Terminology**](50-specifications/10-terminology.md) — Unified terminology definitions
- [**Changelog**](../CHANGELOG.md) — Version update history
- [**License**](../agentrt/LICENSE) — Full text of AGPL v3 + Apache 2.0 dual license

---

## 🔍 Documentation Usage Tips

### Search

Use `Ctrl+F` or `Cmd+F` to search for keywords on the current page.

To search across pages, use the following command:

```bash
# Search for "IPC" in all Markdown files
grep -r "IPC" docs/ --include="*.md"
```

### Documentation Feedback

Found a documentation error or have an improvement suggestion?

1. Click the **Edit this page** button in the upper right corner of the corresponding document page
2. Modify and submit a PR directly
3. Or report it via [AtomGit Issues](https://atomgit.com/openairymax/docs/issues)

### Version Selection

This documentation is always synchronized with the latest stable code. To view historical version documentation:

```bash
# Switch to the documentation of a specific version
git checkout v1.0.0 -- docs/
```

---

## 📊 Documentation Statistics

| Category | Document Count | Estimated Word Count | Last Updated |
|----------|----------------|----------------------|--------------|
| Basic Theories | 9 docs | ~35,000 words | 2026-04-09 |
| Guides | 7 docs | ~25,000 words | 2026-04-09 |
| Architecture | 6 docs | ~40,000 words | 2026-04-09 |
| API Reference | 11 docs | ~45,000 words | 2026-04-09 |
| Development | 10 docs | ~40,000 words | 2026-04-09 |
| Operations | 6 docs | ~30,000 words | 2026-04-09 |
| Troubleshooting | 3 docs | ~12,000 words | 2026-04-09 |
| Specifications | 14 docs | ~55,000 words | 2026-04-09 |
| White Paper & Templates | 4 docs | ~15,000 words | 2026-04-09 |
| References | 3 docs | ~8,000 words | 2026-04-09 |

**Total**: 73 documents, approximately 305,000 words

---

## 📂 Documentation Directory Structure

```
docs/
├── 00-architectural-principles.md   # Five-dimensional orthogonal design principles
├── 00-basic-theories/               # Basic theories (CN/EN bilingual)
│   ├── 01-mcis.md                   # Multibody Cybernetic Intelligent System (EN)
│   ├── 01-mcis-cn.md                # 体系并行论 (CN)
│   ├── 02-cognition-design.md       # Cognition layer design (EN)
│   ├── 02-cognition-design-cn.md    # 认知层设计 (CN)
│   ├── 03-memory-design.md          # Memory layer design (EN)
│   ├── 03-memory-design-cn.md       # 记忆层设计 (CN)
│   ├── 04-design-principles.md      # System design principles (EN)
│   └── 04-design-principles-cn.md   # 系统设计原则 (CN)
├── 10-architecture/                 # Architecture design
│   ├── 01-system-architecture.md    # System overview and layered architecture
│   ├── 02-coreloopthree.md          # Three-layer cognition loop
│   ├── 03-memoryrovol.md            # Memory rovol system
│   ├── 04-microcorert.md            # MicroCore (MicroCoreRT) architecture
│   ├── 05-syscall.md                # Syscall architecture
│   ├── 20-engineering/              # Engineering practices
│   │   ├── 01-testing.md            # Testing architecture
│   │   └── 02-toolkit.md            # Toolchain design
│   ├── 30-kernel/                   # Kernel subsystem
│   │   ├── 01-ipc.md                # IPC communication mechanism
│   │   └── 02-c-language-boundary.md # C language boundary
│   ├── 40-philosophy/               # Design philosophy
│   │   └── 01-design-philosophy.md  # MCIS and five-dimensional design
│   ├── 50-services/                 # Service subsystem
│   │   ├── 01-daemon.md             # Daemon user-space service
│   │   └── 02-logging.md            # Logging system architecture
│   └── 60-diagrams/                 # Architecture diagrams (drawio)
│       ├── 01-coreloopthree-flow.drawio
│       ├── 02-memoryrovol-layers.drawio
│       └── 03-overall-architecture.drawio
├── 10-terminology.md                # Unified terminology
├── 30-api/                          # API reference
│   ├── README.md
│   ├── 01-doxygen-guide.md          # Doxygen documentation guide
│   ├── 02-api-reference.md          # Gateway API reference
│   ├── 03-cli-reference.md          # CLI command reference
│   ├── 10-algorithms/               # Core algorithms
│   │   └── README.md
│   ├── 20-core/                     # Core API
│   │   └── 01-coreloop-api.md       # CoreLoop API
│   ├── 30-daemon/                   # Daemon API
│   │   ├── 01-api-documentation.md  # Daemon API documentation
│   │   └── 02-gateway-api.md        # Gateway API
│   ├── 40-docker/                   # Docker integration
│   │   └── README.md
│   ├── 50-examples/                 # Examples
│   │   └── 01-quickstart.md         # Quick start guide
│   ├── 60-syscalls/                 # Syscall API
│   │   ├── 01-agent.md
│   │   ├── 02-memory.md
│   │   ├── 03-session.md
│   │   ├── 04-skill.md
│   │   ├── 05-task.md
│   │   └── 06-telemetry.md
│   └── 70-toolkit/                  # Multi-language SDK
│       ├── 01-protocol-guide.md     # Protocol guide
│       ├── 02-protocol-quickstart.md # Protocol quickstart
│       ├── 10-go/README.md
│       ├── 20-python/README.md
│       ├── 30-rust/README.md
│       └── 40-typescript/README.md
├── 50-specifications/               # Specifications and contracts
│   ├── README.md
│   ├── 10-contracts/                # Contract specifications
│   │   ├── 01-agent-contract.md
│   │   ├── 02-skill-contract.md
│   │   ├── 03-protocol-contract.md
│   │   ├── 04-syscall-api-contract.md
│   │   ├── 05-logging-format.md
│   │   ├── 06-glossary-index.md
│   │   ├── 01-agent-contract-schema.json
│   │   └── 02-skill-contract-schema.json
│   ├── 20-are-standards/            # ARE Standards (L1-L3)
│   │   ├── README.md
│   │   ├── 01-l1-runtime-interface.md
│   │   ├── 02-l2-service-protocol.md
│   │   └── 03-l3-security-governance.md
│   ├── 30-coding-standard/          # Coding standards
│   │   ├── 01-c-coding-style.md
│   │   ├── 02-c-cpp-secure-coding.md
│   │   ├── 03-code-comment-template.md
│   │   ├── 04-config-audit-log.md
│   │   ├── 05-cpp-coding-style.md
│   │   ├── 06-go-coding-style.md
│   │   ├── 07-go-secure-coding.md
│   │   ├── 08-javascript-coding-style.md
│   │   ├── 09-java-secure-coding.md
│   │   ├── 10-log-standard.md
│   │   ├── 11-naming-conventions.md
│   │   ├── 12-python-coding-style.md
│   │   ├── 13-rust-coding-style.md
│   │   ├── 14-rust-secure-coding.md
│   │   └── 15-security-design.md
│   ├── 40-error-code/               # Error code standard
│   │   └── README.md
│   ├── 50-integration/              # Integration standards
│   │   ├── README.md
│   │   ├── 01-integration-standard.md
│   │   ├── 02-ecosystem-partnership.md
│   │   └── 03-standards-contribution.md
│   ├── 60-ipc/                      # IPC standard
│   │   └── README.md
│   ├── 70-project-erp/              # Project management
│   │   ├── 01-sbom.md
│   │   ├── 02-error-code-reference.md
│   │   ├── 03-manuals-module-requirements.md
│   │   └── 04-resource-management-table.md
│   ├── 80-rpc-api/                  # RPC API standard
│   │   └── README.md
│   ├── 90-sdk/                      # SDK standard
│   │   └── README.md
│   └── 95-service-discovery/        # Service discovery standard
│       └── README.md
├── 60-guides/                       # Getting started and ops guides
│   ├── 01-getting-started.md
│   ├── 02-build-guide.md
│   ├── 03-deployment-guide.md
│   ├── 04-configuration-guide.md
│   ├── 05-config-change-process.md
│   ├── 06-config-drift-detector.md
│   ├── 07-monitoring-guide.md
│   ├── 08-diagnosis-guide.md
│   ├── 09-performance-tuning.md
│   ├── 10-migration-guide.md
│   ├── 11-backup-recovery.md
│   ├── 12-security-gateway.md
│   ├── 13-security-hardening.md
│   ├── 14-testing-guide.md
│   ├── 15-testing-standards.md
│   ├── 16-ci-cd-pipelines.md
│   ├── 17-plugin-sdk-tutorial.md
│   ├── 18-protocol-integration.md
│   ├── 19-prompt-engineering.md
│   ├── 20-manager-development.md
│   ├── 21-best-practices.md
│   ├── 22-known-issues.md
│   ├── 23-troubleshooting-faq.md
│   ├── 24-create-agent.md
│   ├── 25-create-skill.md
│   └── 26-coreloopthree-dag-integration.md
├── 90-references/                   # Other resources
│   └── README.md
├── README.md                        # English documentation entry (this file)
└── README_zh.md                     # Chinese documentation entry
```

---

## 🎯 Documentation Quality Standards

Airymax documentation follows the **Perfectionism Principle (A-4)**:

✅ **Completeness**: Every public API is documented  
✅ **Accuracy**: Example code is runnable, configuration parameters are verified  
✅ **Timeliness**: Documentation is updated within 24 hours of code changes  
✅ **Readability**: Clear language is used, avoiding excessive technical jargon  
✅ **Actionability**: Every guide provides step-by-step instructions  

---

## 📞 Contact


---

**© 2025-2026 SPHARX Ltd. All Rights Reserved.**
