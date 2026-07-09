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

- [**MCIS (Multi-body Control Intelligence System)**](Basic_Theories/EN_01_MCIS.md) — Multi-body Control Intelligence System ([中文](Basic_Theories/CN_01_体系并行.md))
- [**Cognition Layer Design**](Basic_Theories/EN_02_Cognition_Theory.md) — Dual thinking system and three-layer architecture ([中文](Basic_Theories/CN_02_认知层设计.md))
- [**Memory Layer Design**](Basic_Theories/EN_03_Memory_Theory.md) — Four-layer memory rovol model ([中文](Basic_Theories/CN_03_记忆层设计.md))
- [**Design Principles**](Basic_Theories/EN_04_Design_Principles.md) — Introduction to the five-dimensional orthogonal design principles ([中文](Basic_Theories/CN_04_系统设计原则.md))
- [**Design Philosophy**](Capital_Architecture/philosophy/design_philosophy.md) — Detailed exposition of MCIS and the five-dimensional orthogonal design system
- [**Architectural Principles (Complete)**](ARCHITECTURAL_PRINCIPLES.md) — Detailed exposition of the 24 S/K/C/E/A principles

---

### 2️⃣ Guides

Guided tutorials for new users to get started quickly:

- [**Quick Start**](Capital_Guides/getting_started.md) — From zero to Hello World in 5 minutes
- [**Configuration Guide**](Capital_Guides/configuration_guide.md) — Complete description of configuration options
- [**Deployment Guide**](Capital_Guides/deployment_guide.md) — Production deployment best practices (Docker/K8s/Monitoring/Security)
- [**Create Agent**](Capital_Guides/create_agent.md) — Complete Agent development workflow
- [**Create Skill**](Capital_Guides/create_skill.md) — Complete Skill development workflow
- [**Plugin SDK Tutorial**](Capital_Guides/plugin_sdk_tutorial.md) — Complete Plugin SDK development guide
- [**Prompt Engineering Guide**](Capital_Guides/prompt_engineering.md) — Full lifecycle of prompt templates/injection/tuning
- [**Migration Guide**](Capital_Guides/migration_guide.md) — Version upgrades and data migration
- [**Build Guide**](Capital_Guides/build_guide.md) — Build system and compilation options
- [**Testing Standards**](Capital_Guides/testing_standards.md) — Test layering and coverage standards
- [**Performance Tuning**](Capital_Guides/performance_tuning.md) — Kernel parameters and performance optimization
- [**Protocol Integration**](Capital_Guides/protocol_integration.md) — Multi-protocol adaptation and routing

---

### 3️⃣ Architecture

In-depth understanding of Airymax's design philosophy and technical implementation:

- [**MicroCore Architecture**](Capital_Architecture/microcorert.md) — Implementation of K-1~K-4 principles
- [**CoreLoopThree**](Capital_Architecture/coreloopthree.md) — Cognition→Execution→Memory
- [**Memory Rovol System**](Capital_Architecture/memoryrovol.md) — L1→L2→L3→L4 four-layer memory
- [**IPC Communication**](Capital_Architecture/kernel/ipc.md) — Binder/Channel/Buffer inter-process communication
- [**Syscall Architecture**](Capital_Architecture/syscall.md) — Unified interface between user space and kernel space
- [**Logging System Architecture**](Capital_Architecture/services/logging.md) — Cross-language observability and dynamic feedback regulation
- [**Architecture Overview**](Capital_Architecture/architecture.md) — System overview and layered architecture diagram
- [**C Language Boundary**](Capital_Architecture/kernel/c_language_boundary.md) — C core responsibility scope and FFI boundary

---

### 4️⃣ API Reference

Complete interface documentation, including request examples and response formats:

**Syscall API**:

- [**API Overview**](Capital_API/README.md) — API hierarchy and design philosophy
- [**Gateway API Reference**](Capital_API/api-reference.md) — HTTP/WebSocket endpoint API reference
- [**CLI Command Reference**](Capital_API/cli-reference.md) — Complete commands for the unified CLI tool
- [**Task Management API**](Capital_API/syscalls/task.md) — submit/query/wait/cancel full task lifecycle
- [**Memory Management API**](Capital_API/syscalls/memory.md) — write/query/evolve/forget four-layer memory operations
- [**Session Management API**](Capital_API/syscalls/session.md) — create/get/close/list session management
- [**Observability API**](Capital_API/syscalls/telemetry.md) — metrics/traces telemetry interface
- [**Agent Management API**](Capital_API/syscalls/agent.md) — spawn/terminate/invoke Agent management

**Multi-language SDK**:

- [**Python SDK**](Capital_API/toolkit/python/README.md) — Python language binding API
- [**Go SDK**](Capital_API/toolkit/go/README.md) — Go language binding API
- [**Rust SDK**](Capital_API/toolkit/rust/README.md) — Rust language binding API
- [**TypeScript SDK**](Capital_API/toolkit/typescript/README.md) — TypeScript language binding API

**Core Algorithms**:

- [**Algorithm Implementation Docs**](Capital_API/algorithms/README.md) — Document processing/search index/quality validation/performance optimization core algorithms

---

### 5️⃣ Development

Essential knowledge for contributing to Airymax development:

**Contribution & Testing**:

- [**Contribution Guide**](../agentrt/CONTRIBUTING.md) — Complete PR submission workflow
- [**Testing Guide**](Capital_Guides/testing_guide.md) — Unit tests, integration tests, E2E tests

**Coding Standards**:

- [**C Coding Style Standard**](Capital_Specifications/coding_standard/C_coding_style_standard.md) — C naming/functions/error handling/concurrency/security standards
- [**C++ Coding Style Standard**](Capital_Specifications/coding_standard/Cpp_coding_style_standard.md) — C++ coding standards
- [**Python Coding Style Standard**](Capital_Specifications/coding_standard/Python_coding_style_standard.md) — Python type design/async/error handling standards
- [**JavaScript Coding Style Standard**](Capital_Specifications/coding_standard/JavaScript_coding_style_standard.md) — JavaScript/TypeScript coding standards
- [**Naming Conventions**](Capital_Specifications/coding_standard/NAMING_CONVENTIONS.md) — Component/file/function/type/constant naming conventions
- [**Code Comment Template**](Capital_Specifications/coding_standard/Code_comment_template.md) — Doxygen/docstring comment standards
- [**Logging Standard**](Capital_Specifications/coding_standard/Log_standard.md) — Log level/content/format/quality standards

**Secure Coding**:

- [**Security Design Standard**](Capital_Specifications/coding_standard/Security_design_standard.md) — D1~D4 four-layer protection/encryption/authentication/privacy protection
- [**C/C++ Secure Coding Standard**](Capital_Specifications/coding_standard/C_Cpp_secure_coding_standard.md) — C/C++ secure coding practices
- [**Java Secure Coding Standard**](Capital_Specifications/coding_standard/Java_secure_coding_standard.md) — Java secure coding practices

---

### 6️⃣ Operations

Operational assurance for production environments:

- [**Docker Deployment**](../Docker/README.md) — Complete containerized deployment solution
- [**Monitoring & Ops**](Capital_Guides/monitoring_guide.md) — Prometheus+Grafana monitoring stack
- [**Backup & Recovery**](Capital_Guides/backup_recovery.md) — Data backup and disaster recovery
- [**Kernel Tuning**](Capital_Guides/performance_tuning.md) — Kernel parameter tuning and performance optimization

---

### 7️⃣ Troubleshooting

Common issues and solutions:

- [**FAQ**](Capital_Guides/troubleshooting_faq.md) — Troubleshooting and high-frequency issues
- [**Error Diagnosis**](Capital_Guides/diagnosis_guide.md) — Log analysis and problem localization
- [**Known Issues**](Capital_Guides/known_issues.md) — Known bugs and temporary workarounds

---

### 8️⃣ Specifications

The standardized specification system for the Airymax project:

**Contract Specifications**:

- [**Contract Overview**](Capital_Specifications/agentrt_contract/glossary_index.md) — Glossary and quick index
- [**Agent Contract**](Capital_Specifications/agentrt_contract/agent_contract.md) — Agent capability description specification ([Schema](Capital_Specifications/agentrt_contract/agent_contract_schema.json))
- [**Skill Contract**](Capital_Specifications/agentrt_contract/skill_contract.md) — Skill capability description specification ([Schema](Capital_Specifications/agentrt_contract/skill_contract_schema.json))
- [**Protocol Specification**](Capital_Specifications/agentrt_contract/protocol_contract.md) — HTTP/WS/Stdio gateway + JSON-RPC 2.0
- [**Syscall API Specification**](Capital_Specifications/agentrt_contract/syscall_api_contract.md) — Syscall interface contract
- [**Logging Format Specification**](Capital_Specifications/agentrt_contract/logging_format.md) — Structured JSON log format

**Integration Standards**:

- [**Integration Standards Overview**](Capital_Specifications/integration_standards/README.md) — Index of inter-module integration standards
- [**Manager Configuration Integration Standard**](Capital_Specifications/integration_standards/INTEGRATION_STANDARD.md) — Manager module integration with the unified configuration library

**Project Management**:

- [**Error Code Reference**](Capital_Specifications/project_erp/error_code_reference.md) — Complete error code definitions and handling recommendations
- [**Resource Management Table**](Capital_Specifications/project_erp/resource_management_table.md) — Resource creation/release/ownership specifications
- [**Software Bill of Materials (SBOM)**](Capital_Specifications/project_erp/SBOM.md) — Component/dependency/license/security information
- [**Module Requirements**](Capital_Specifications/project_erp/manuals_module_requirements.md) — manuals module requirements and technical specifications

**Terminology**:

- [**Unified Terminology**](Capital_Specifications/TERMINOLOGY.md) — Unified Airymax terminology definitions

---

### 9️⃣ White Paper & Templates

- [**Technical White Paper**](White_Paper/README.md) — Official Airymax technical white paper (CN/EN)
- [**Document Template**](Quote_Templates/_template.md) — General document template
- [**API Document Template**](Quote_Templates/_template_api.md) — API document authoring template
- [**Guide Document Template**](Quote_Templates/_template_guide.md) — Guide document authoring template

---

### 🔟 References

Supplementary materials and external links:

- [**Unified Terminology**](Capital_Specifications/TERMINOLOGY.md) — Unified terminology definitions
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
├── ARCHITECTURAL_PRINCIPLES.md    # Complete five-dimensional orthogonal design principles
├── README.md                      # English documentation entry (this file)
├── README_zh.md                   # Chinese documentation entry
├── TERMINOLOGY.md                 # Unified terminology
├── Basic_Theories/                # Basic theories (CN/EN bilingual)
│   ├── CN_01_体系并行.md
│   ├── CN_02_认知层设计.md
│   ├── CN_03_记忆层设计.md
│   ├── CN_04_系统设计原则.md
│   ├── EN_01_MCIS.md
│   ├── EN_02_Cognition_Theory.md
│   ├── EN_03_Memory_Theory.md
│   └── EN_04_Design_Principles.md
├── Capital_Architecture/          # Architecture design
│   ├── architecture.md            # System overview and layered architecture diagram
│   ├── microcorert.md             # MicroCore (MicroCoreRT) architecture details
│   ├── coreloopthree.md           # Three-layer cognition loop
│   ├── memoryrovol.md             # Memory rovol system
│   ├── syscall.md                 # Syscall architecture
│   ├── kernel/                    # Kernel subsystem
│   │   ├── ipc.md                 # IPC communication mechanism
│   │   └── c_language_boundary.md # C language boundary definition
│   ├── services/                  # Service subsystem
│   │   ├── daemon.md              # Daemon user-space service
│   │   └── logging.md             # Logging system architecture
│   ├── engineering/               # Engineering practices
│   │   ├── testing.md             # Testing architecture
│   │   └── toolkit.md             # Toolchain design
│   ├── philosophy/                # Design philosophy
│   │   └── design_philosophy.md   # MCIS and five-dimensional orthogonal design system
│   └── diagrams/                  # Architecture diagrams (drawio)
├── Capital_API/                   # API reference
│   ├── README.md
│   ├── api-reference.md           # Gateway API reference manual
│   ├── cli-reference.md           # CLI command reference manual
│   ├── syscalls/                  # Syscall API
│   │   ├── task.md
│   │   ├── memory.md
│   │   ├── session.md
│   │   ├── telemetry.md
│   │   └── agent.md
│   ├── toolkit/                   # Multi-language SDK
│   │   ├── python/README.md
│   │   ├── go/README.md
│   │   ├── rust/README.md
│   │   └── typescript/README.md
│   └── algorithms/                # Core algorithms
│       └── README.md
├── Capital_Guides/                # Getting started and ops guides
│   ├── backup_recovery.md
│   ├── best_practices.md
│   ├── build_guide.md
│   ├── ci_cd_pipelines.md
│   ├── config_change_process.md
│   ├── config_drift_detector.md
│   ├── configuration_guide.md
│   ├── create_agent.md
│   ├── create_skill.md
│   ├── deployment_guide.md
│   ├── diagnosis_guide.md
│   ├── getting_started.md
│   ├── known_issues.md
│   ├── manager_development.md
│   ├── migration_guide.md
│   ├── monitoring_guide.md
│   ├── performance_tuning.md
│   ├── plugin_sdk_tutorial.md
│   ├── prompt_engineering.md
│   ├── protocol_integration.md
│   ├── security_gateway.md
│   ├── security_hardening.md
│   ├── testing_guide.md
│   ├── testing_standards.md
│   └── troubleshooting_faq.md
├── Capital_Specifications/         # Specifications and contracts
│   ├── README.md
│   ├── agentrt_contract/          # Contract specifications
│   │   ├── agent_contract.md
│   │   ├── agent_contract_schema.json
│   │   ├── skill_contract.md
│   │   ├── skill_contract_schema.json
│   │   ├── protocol_contract.md
│   │   ├── syscall_api_contract.md
│   │   ├── logging_format.md
│   │   └── glossary_index.md
│   ├── coding_standard/           # Coding standards
│   │   ├── NAMING_CONVENTIONS.md  # Naming conventions
│   │   ├── C_coding_style_standard.md
│   │   ├── Cpp_coding_style_standard.md
│   │   ├── Python_coding_style_standard.md
│   │   ├── JavaScript_coding_style_standard.md
│   │   ├── C_Cpp_secure_coding_standard.md
│   │   ├── Java_secure_coding_standard.md
│   │   ├── Security_design_standard.md
│   │   ├── Log_standard.md
│   │   └── Code_comment_template.md
│   ├── integration_standards/     # Integration standards
│   │   ├── README.md
│   │   └── INTEGRATION_STANDARD.md
│   └── project_erp/              # Project management
│       ├── SBOM.md
│       ├── error_code_reference.md
│       ├── manuals_module_requirements.md
│       └── resource_management_table.md
├── Source_Other/                  # Other resources
│   └── Airymax-desktop-preview.gif
├── White_Paper/                   # White paper
│   ├── README.md
│   └── history/
└── Quote_Templates/               # Document templates
    ├── _template.md
    ├── _template_api.md
    └── _template_guide.md
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
