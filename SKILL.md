---
name: deliver-with-subagents
description: 用于已确认需求、OpenSpec change 或 tickets 的隔离子会话开发交付；不用于需求发现、方案讨论或范围澄清。包含测试、独立 Review 与修复闭环。
---

# Deliver with Subagents

只在需求与验收边界已经确认后使用。主会话负责锁定范围、调度、状态与最终结论；Dev、Integrator、Reviewer 使用不继承完整历史的全新子会话，只接收完成当前职责所需的精简简报。

## 不可变门禁

- 若业务规则、数据、权限、安全、API 契约或范围仍有关键歧义，停止受影响节点，返回 OpenSpec、Wayfinder 或用户确认，不把猜测写进生产代码。
- 支持时以 `fork_turns: "none"` 启动 fresh Dev、Integrator、Reviewer；主会话只保留决策、状态、固定 SHA、证据摘要和未决项。
- 同一工作区同一时刻只有一个 writer。纯串行任务由唯一 Dev 直接写目标工作区，不创建 Integrator；只有并行批次才由唯一 Integrator 写目标工作区。Reviewer 始终只读。
- 写入任务默认串行。只有依赖、写入文件、共享契约、数据库/fixture/端口等资源、集成与回滚都能证明独立时才并行；“不同模块”或“已有 checkpoint”本身不构成证明。
- 默认每阶段创建本地 checkpoint commit，但不等于发布授权。除非用户明确要求，不 push、不建 PR、不部署、不归档，也不 rebase、squash、reset 或清理 checkpoint。
- Dev 自检不能替代独立 Review。所有成立且可执行的 P0-P3 问题都要修复并对新 fixed point 复查。

## 1. 锁定输入

1. 按“用户当前指令 > 已确认 spec/tickets > 项目规则 > 现有行为”确定范围；记录验收标准、不做内容、风险、测试与发布边界。
2. 有 OpenSpec 时先验证 change。只有当前任务授权维护 artifact 或项目流程明确要求时，才修复机械性的结构错误；任何语义、范围、验收标准变化都必须先确认。仅在被授权时更新 task 状态。
3. 在生成 DAG 或派发任何 writer 前，完整读取 [checkpoint-and-recovery.md](references/checkpoint-and-recovery.md)，检查 dirty 状态并固定 `COMMIT_MODE`、任务前基线和恢复方式；再记录用户指定的单任务、串行/并行、环境、真实接口与时限要求。
4. 形成精简交付简报，禁止把整段主会话历史复制给子会话。未指定的行为使用本 Skill 默认值。

完成标准：需求输入、验收边界、发布权限和待确认项足以让 Dev 独立执行。

## 2. 拆分与调度

1. 用户指定只用一个 Dev、不要拆分或不要并行时直接遵循。目标紧密耦合时也使用一个 Dev。
2. 只有存在可独立验收的阶段或单个上下文难以稳定容纳时才拆分；优先复用已确认的 OpenSpec/Wayfinder 阶段，不拆无独立价值的机械步骤。
3. 共享文件、未冻结 API/DTO/schema/核心抽象、共享数据库或 fixture、迁移顺序、不可隔离环境任一存在时，建立顺序边。先串行冻结共享契约，再重新评估后续任务。
4. 对满足全部并行白名单的 ready 节点，可用隔离 worktree 并行；只读分析、测试设计和只读审计在不争用环境时可更积极并行。worktree 能力失败则保留证据并自动回退串行。
5. 主任务维护依赖 DAG 和 ready queue：一个节点结束后自动启动下一 ready 节点；多个 ready 节点仅在白名单成立时并行。局部 `FAIL`/`BLOCKED` 只阻塞该节点及其依赖。
6. 并行时发现文件或语义重叠，停止后启动的冲突 writer，保存 checkpoint；先集成前一个结果，再从最新目标 checkpoint 串行重启受影响任务，不让主会话硬合并语义冲突。
7. MySQL、Docker、第三方 Test Mode、测试账号和 fixture 仅在已授权、非生产、资源隔离且副作用可回滚时作为环境节点提前并行；环境就绪不等于接口验收通过。

状态含义：

- `DEV_READY`：并行 Dev 已完成实现、自检、focused tests 和 task checkpoint，可交给 Integrator；不能解锁依赖节点。
- `DEV_PASS`：串行阶段完成适用验证和 stage checkpoint；并行批次还必须完成集成验证、batch checkpoint 及强制 batch `REVIEW_PASS`。只有达到该状态才解锁依赖节点。
- `REVIEW_PASS`：独立 Reviewer 对固定 `code_checkpoint` 与范围复查后，没有已知且可执行的 P0-P3 问题。
- HTTP acceptance 状态：`HTTP_PASS`、`HTTP_FAIL`、`HTTP_BLOCKED`、`HTTP_STALE` 或 `HTTP_NOT_APPLICABLE`；它与代码、Review 状态分别记录。
- `FAIL` / `BLOCKED`：实现或验证失败，或缺少继续所需的权限、环境、凭据、fixture、用户决定。
- `STOPPED_INCOMPLETE`：达到时限后收口，仍有未完成项；它不是通过或完成状态。

## 3. 开发、测试与 checkpoint

在首次派发子会话、独立 Review、真实 HTTP 验证或最终汇报前，完整读取 [evidence-and-briefs.md](references/evidence-and-briefs.md)。

Dev 必须：

- 完成最小生产可用实现，不扩大范围，不修改无关 contract；只在兼容且可用时调用其他 Skill，不因缺失而降低交付标准。
- 测试缝和预期行为清晰时使用 `$tdd`；Bug 修复优先先写能复现问题的回归测试。测试缝会改变设计时先确认，否则按最小实现加事后测试并说明原因。
- 开发中跑 focused tests。实现后用 `$code-simplifier` 或等价规则简化本次改动，补充只解释关键“为什么”的注释，重跑受影响测试，再做需求遗漏、异常路径、测试缺口与复杂度自检。
- 串行阶段收口或并行批次集成后只运行一次适用的受影响回归、类型检查、编译或构建；最终验证只补尚未覆盖的整体验证，避免机械重复重测试。

新增或修改 HTTP API 时，建立以固定 code checkpoint 和可用环境为输入的独立 acceptance 节点；交付完成前必须取得对最终 fixed point 仍有效的 `HTTP_PASS`。真实调用、证据、阻塞、wire type、凭据和过期处理遵循 [evidence-and-briefs.md](references/evidence-and-briefs.md)；HTTP 节点状态不抹除已取得的代码或 Review 状态，但会约束显式依赖节点与整体完成。

## 4. 独立 Review 与修复

1. 单任务或纯串行链只在全部实现与可执行自动化测试完成后做一次最终独立 Review；HTTP acceptance 状态单独记录，不阻止代码 Review 形成独立结论。
2. 并行任务必须在 Integrator 合并、批次验证和 batch checkpoint 后做独立 Review；该 Review/fix loop 通过后批次才达到 `DEV_PASS` 并解锁后继。只有资金、权限、安全、迁移、并发、共享核心契约、难回滚副作用或大型子任务才增加合并前 Review。
3. 最后一次批次 Review 明确覆盖全部累计 diff 与整体调用链时，可同时作为最终 Review。
4. 优先使用只读 custom Reviewer，并在 fixed point 明确时使用 `$code-review` 或等价审查。不可用时，Review 前后核对规范化 workspace diff/内容指纹；发生非预期变化则该 Review 无效。
5. 严重度：P0 为灾难性数据、安全或全局可用性事故；P1 为阻断核心需求、主流程或发布；P2 为重要场景的真实正确性、兼容性或可靠性问题；P3 为低影响但可复现、应在当前范围修复的局部缺陷。纯风格偏好或无失败路径的建议不是 P3。
6. 成立问题交给唯一 writer：单任务交回原 Dev，串行跨阶段问题交给当前目标 Dev，合并前问题交给对应并行 Dev，批次/跨任务/整体问题交给 Integrator。修复后重跑相关测试、创建新 checkpoint，并让独立 Reviewer 审查新的 fixed point，禁止在旧结论上口头关闭。

## 5. 时限与完成

- 用户可指定总时限、阶段时限或不限时。指定时限是硬上限；调度时固定绝对 `deadline_at`，每个子会话使用“用户总时限剩余值”和当前节点时限中的较早者，禁止自行重置。
- 未指定时，每个 Dev 的收口时钟从第一版功能实现完成起算，每个并行批次时钟从 Integrator 收到首个固定 `TASK_SHA` 起算，最终时钟从全部实现节点已通过或阻塞并进入整体收口起算，各默认 60 分钟墙钟时间；等待和重试不暂停、不重置。长 DAG 没有额外的默认总时限；最后批次 Review 兼作最终 Review 时只使用一个时钟。
- 不启动明显无法在剩余时间内安全结束的操作。到达 deadline 后停止所有可安全中断的工作，只允许完成保护数据/安全或保存可恢复 checkpoint 所必需的清理，并标记 `STOPPED_INCOMPLETE`。超时后不能新授予 pass 或“已完成”；超时前已取得的状态保留为历史证据，但不改变整体未完成结论。
- 只有所有范围内节点完成、必需自动化通过、HTTP 不适用或 `HTTP_PASS` 对最终 fixed point 仍有效，且 `REVIEW_PASS` 对最终 fixed point 仍有效、没有已知可执行 P0-P3 时，才能声明交付完成。
- 最终只汇总：范围与节点状态、关键文件、自动化及 HTTP 证据、checkpoint、Review 与修复、P0-P3、未完成/风险、commit/push/deploy/archive 状态。明确区分代码完成、测试通过、接口验收和上线完成。
