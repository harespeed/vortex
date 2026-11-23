# Vortex Kernel Docs

本目录是 **Vortex** 的设计与运维文档。**Vortex 只是 kernel**：不是平台、不是扩展、不是终端产品。**当前阶段只做设计与验证，不写实现代码。**

- Kernel：本仓库（运行时、端口契约、策略、状态、扩展监督）
- Extensions：`ext-software` / `ext-os` / `ext-hardware`，在独立仓库
- Product：家庭产品是另一个项目，不在本仓库

## 文档地图

| 文档 | 内容 | 读者 |
|---|---|---|
| [scope.md](scope.md) | **项目范围：Vortex 只写 kernel**；内/外范围、仓库结构、完成定义、边界规则 | 所有人 |
| [implementation-plan.md](implementation-plan.md) | **实施方案**：里程碑、WBS、人日、验收、测试、风险、决策门 | 所有人 |
| [architecture.md](architecture.md) | 目标、约束、进程/线程模型、状态机、安全架构、供应链边界、CVE 结论、ADR | 架构/实现者 |
| [howto.md](howto.md) | 从零构建 OpenSSL/SQLite、TLS、PQC 信封加密、沙箱、备份、CVE 监控 | 运维/实现者 |
| [robotics.md](robotics.md) | 具身扩展：云端认知 / 边缘技能 / MCU 实时，技能契约与安全包络 | 机器人/安全 |
| [hardware.md](hardware.md) | 电机、铰链、执行器、驱动器、电子 IO、台架与首次上电 | 硬件/机械 |
| [world-interface.md](world-interface.md) | 物理世界接口：传感器→符号化状态→LLM；指代消解、接地、不确定性、时间新鲜度 | Agent/感知 |
| [knowledge-layer.md](knowledge-layer.md) | **Kernel 知识层**：开放世界状态、OWL-compatible schema、RDFS/OWL RL 子集、provenance、belief revision | 架构/知识 |
| [kernel-ports.md](kernel-ports.md) | 最小内核与端口/适配器：母头契约、进程边界、版本、权限；扩展分 software/os/hardware 三类 | 架构/扩展开发 |
| [general-agentic-system-design.md](general-agentic-system-design.md) | 通用 agentic system 的设计方案（为什么是单核 + 图壳 + 持久化） | 产品/架构 |
| [2026-agent-framework-architecture-report.md](2026-agent-framework-architecture-report.md) | 2026 年框架与架构的市场/数据全景 | 研究/选型 |

## 当前决策（一句话版）

**本仓库只实现 kernel 层。** C++20 单二进制、单进程、SQLite 状态、子进程沙箱、OpenSSL 3.5.10+ 提供 TLS 1.3 与 PQC 原语；零包管理器，只允许 OpenSSL 与 SQLite amalgamation 两个可审计依赖。所有 `ext-software` / `ext-os` / `ext-hardware` 在外部仓库实现。知识层属于 kernel（开放世界 belief store + RDFS/OWL 2 RL 子集）；完整 OWL EL/DL 推理器走可选 `OntologyPort`。

## 明确不做

- 不做多租户 SaaS、Kubernetes、计费、控制面/数据面拆分。
- 不做自由多 agent 群聊、角色扮演编排。
- 不引入 pip/npm/vcpkg/conan、模型 SDK、MCP 第三方 server、CDN 资源。
- 不自己实现 TLS、AEAD、KDF、ML-KEM/ML-DSA。

## 状态

| 项 | 状态 |
|---|---|
| 架构与威胁模型 | 完成初稿 |
| CVE 基线 | 已核对（2026-11-20） |
| 构建与部署步骤 | 完成初稿（命令标注“目标/待实现”） |
| 具身扩展（机器人） | 完成初稿 |
| 实现代码 | 未开始 |
