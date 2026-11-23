# 项目范围：Vortex 只写 Kernel

> 状态：设计稿。没有实现代码。  
> **Vortex = kernel。** 本仓库只实现 **agent kernel**：运行时、端口契约、策略、状态、事件、扩展监督、参考仿真适配器。  
> 所有真实设备/系统扩展（`ext-software`、`ext-os`、`ext-hardware`）在**其他仓库**实现。

---

## 1. 一句话范围

**Kernel 是插座标准 + 最小运行时，不是电器大全。**  
本仓库交付“怎么安全地跑、怎么恢复、怎么授权、怎么扩展”，不交付“怎么控制某台车/机械臂/桌面”。

---

## 2. 在本仓库内（In Scope）

| 模块 | 内容 | 备注 |
|---|---|---|
| Run 运行时 | Run/Step 生命周期、检查点、恢复、取消、预算 | 核心 |
| 状态与事件 | `runs/steps/events/approvals/artifacts` | SQLite |
| 端口契约 | Model/Tool/Event/Store/Sensor/Actuator/Skill/World | schema + version |
| 策略与审批 | 角色、工具/技能授权、不可逆操作确认 | 默认拒绝 |
| 扩展监督 | 发现、握手、健康、隔离、重启、版本协商 | 独立进程 |
| 进程协议 | UDS + 帧协议 + 心跳 + 幂等/取消 | 无动态库 |
| PWI 契约 | 状态快照、事件、指代消解、接地校验的接口 | 实现可后置 |
| 模型/工具路由 | 统一模型网关、工具注册表 | 不绑定厂商 |
| 审计与观测 | 结构化日志、trace、成本 | 最小集 |
| 最小控制 API | HTTP/SSE：提交 run、查状态、审批、取消 | 不含 UI |
| CLI | `vortexd`、`vortexctl`、`vortex-conformance` | 运维/测试 |
| 参考适配器 | `ext-sim`：唯一在仓库内的适配器 | 用于契约测试 |
| 测试 | 单元、契约、故障注入、恢复 | 必须可重复 |

---

## 3. 不在本仓库（Out of Scope）

| 内容 | 归属 |
|---|---|
| `ext-software`：http/sql/files/code/browser/SaaS/LLM 适配 | 独立仓库 |
| `ext-os`：shell/进程/桌面/输入/网络 | 独立仓库 |
| `ext-hardware`：arm/car/home/camera/drone/sensor | 独立仓库 |
| 设备驱动、CAN/串口协议、厂商 SDK | 扩展仓库 |
| 感知、SLAM、标定、运动学/动力学 | 扩展仓库 |
| 机器人技能实现（抓取/导航/力控） | 扩展仓库 |
| UI、手机 App、语音前端 | 独立项目 |
| 模型训练/微调、推理引擎 | 不在范围 |
| 多租户、计费、Kubernetes、控制面/数据面拆分 | 不在范围 |
| ROS 2 / CUDA / TensorRT 集成 | 不在内核 |

**唯一例外**：`ext-sim` 在仓库内，因为它是**测试夹具**，不是设备实现；它必须通过与其他扩展相同的端口契约。

---

## 4. 仓库结构（目标）

```text
kernel/
  CMakeLists.txt
  src/
    kernel/        # run, step, state, checkpoint, budget
    ports/         # 端口 schema、版本、错误模型
    policy/        # 角色、审批、限幅、能力令牌
    event/         # 事件总线、订阅、游标
    store/         # SQLite、artifacts
    ext/           # 扩展监督、握手、帧协议、健康
    pwi/           # 世界状态/指代/接地的接口与默认实现
    api/           # HTTP/SSE 最小控制面
    cli/           # vortexd / vortexctl / conformance
  ext-sim/         # 参考适配器（测试用）
  tests/
    unit/ conformance/ fault/ recovery/
  docs/
```

**没有** `ext-software/`、`ext-os/`、`ext-hardware/` 目录。

---

## 5. Kernel 的“完成”定义

### v0.1（最小可用）

- [ ] 提交 run → 模型调用 → 工具调用 → 完成/失败，全链路可跑。
- [ ] SQLite 持久化；杀进程后从检查点恢复。
- [ ] 策略拒绝未授权工具；预算超限暂停。
- [ ] `vortexctl` 可创建用户、提交 run、查看 trace。
- [ ] `ext-sim` 通过握手与契约测试。

### v0.2（可扩展）

- [ ] 扩展进程协议：握手/心跳/隔离/重启。
- [ ] 版本协商；不匹配拒绝。
- [ ] 契约测试套件：错误码、超时、取消、背压。
- [ ] 审批流：暂停/批准/拒绝/超时。

### v0.3（物理就绪，不实现设备）

- [ ] `SensorPort` / `ActuatorPort` / `SkillPort` 契约冻结草案。
- [ ] `ext-sim` 支持模拟关节/传感器/故障注入。
- [ ] 安全等级 S0–S3 的策略映射。
- [ ] PWI：快照/事件/指代/接地的接口与默认实现。

### v1.0（冻结）

- [ ] 端口有 2+ 独立实现、6 个月使用。
- [ ] 兼容策略、弃用策略、迁移文档。
- [ ] 安全评审：扩展隔离、权限、审计。

---

## 6. 边界判定规则

新增代码前问四个问题：

1. **它认识具体设备/协议吗？** 是 → 扩展仓库。
2. **它依赖厂商 SDK/CUDA/ROS 吗？** 是 → 扩展仓库。
3. **它是策略/契约/运行时原语吗？** 是 → kernel。
4. **两个以上扩展会重复实现它吗？** 是 → 先做成内核端口候选，等 2+ 实现再冻结。

**反例**：

- ❌ 在 kernel 里写 `if device == "car"`。
- ❌ 在 kernel 里解析 CAN 帧。
- ❌ 在 kernel 里做运动学正解。
- ❌ 在 kernel 里加载 `.so` 插件。
- ❌ 把 `ext-sim` 的特例写进核心循环。

---

## 7. 依赖与约束

| 项 | 规则 |
|---|---|
| 语言 | C++20；POSIX |
| 加密 | OpenSSL 3.5.10+（TLS + PQC 原语） |
| 存储 | SQLite 3.54.1+ amalgamation |
| 依赖 | 仅 libc/pthreads + OpenSSL + SQLite；零包管理器 |
| 扩展 | 独立进程；UDS + 帧协议；不加载动态库 |
| 平台 | macOS 优先，Linux 次之；Jetson 作为部署目标，不引入 CUDA |
| UI | 不在内核；内核只提供 HTTP/SSE API |

---

## 8. 与扩展仓库的契约

- Kernel 定义 schema、版本、错误码、生命周期、安全等级。
- 扩展实现端口，声明 `class` / `kernel` 版本 / 权限 / 资源。
- Kernel 不导入扩展代码；只通过进程协议通信。
- 扩展仓库自带契约测试；升级内核前必须跑通。
- Kernel 不保证扩展质量；只保证隔离、权限与审计。

---

## 9. 不做什么（再次强调）

- 不做“通用设备模型”枚举；只做能力描述与三类扩展。
- 不做“未来可能用到”的空端口；按 v0.1→v1.0 逐步冻结。
- 不做 UI/语音/行业协议。
- 不做多租户/计费/云平台。
- 不把测试夹具当产品功能。

**结论**：本仓库只写 kernel。设备、系统、软件扩展都是外部公头；kernel 的交付物是**稳定的母头契约 + 可恢复运行时 + 安全策略 + 扩展监督**。这样汽车、机械臂、桌面、SaaS 都只是外部项目，不需要改内核。
