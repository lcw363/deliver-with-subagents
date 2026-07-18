# subagent-delivery

[中文](./README.md) | [English](./README_EN.md)

用于**需求确认后的开发交付**的 Codex Skill：以 fresh 子会话隔离实现细节，自动创建不推送的阶段 checkpoint，保守调度串并行任务，并完成测试、真实 HTTP 验证、独立 Review 与修复闭环。

## 适用范围

- 输入应是已确认的 OpenSpec change、tickets、PRD tasks 或明确开发需求。
- 本 Skill 不负责需求发现、方案讨论或范围澄清。建议先用当前最强推理档配合 **OpenSpec** 完善需求；大型多阶段任务先用 **Wayfinder** 明确阶段和依赖，再进入本 Skill。
- 仍有影响业务、数据、权限、安全、API 契约或范围的关键歧义时，会停下等待确认。

## 核心保障

- Dev、Integrator、Reviewer 使用不继承完整历史的 fresh 子会话；主会话只保留关键决策、状态、SHA 和证据摘要。
- 模型按角色和风险动态选择：常规 Dev 优先较快的均衡档，Integrator、独立 Reviewer 和高风险/重复失败任务使用当前最强档；不绑定具体模型版本，用户可覆盖。
- 自动选择一个或多个开发子任务，并在节点完成后自动启动下一 ready 节点。
- 写入默认串行。只有依赖、文件、契约、数据库/fixture/端口、集成和回滚均可证明独立时才并行；共享文件、未冻结契约或共享数据资源一律串行。
- 串行任务由唯一 Dev 直接写目标工作区；只有并行批次才使用隔离 worktree 和唯一 Integrator。
- 每个阶段默认自动创建本地 checkpoint commit，但不 push、不建 PR、不部署、不归档；用户或项目禁止 commit 时改用串行 snapshot 模式。
- 条件合适时使用 TDD；实现后简化代码并自检，再进行固定 checkpoint 的独立 Review。成立的 P0-P3 问题会修复并复查。
- 新增或修改 HTTP API 时，用脱敏后的实际代表参数调用真实路由；Mock 和 schema 检查不能替代接口验收。
- 用户可指定时限或不限时。未指定时，每个 Dev 收口、每个并行批次闭环和最终整体收口各默认 60 分钟；超时标记 `STOPPED_INCOMPLETE`，不会伪装成完成。

## 安装与调用

```bash
mkdir -p ~/.agents/skills
git clone https://github.com/lcw363/subagent-delivery.git \
  ~/.agents/skills/subagent-delivery
```

显式调用最可靠：

```text
请使用 $subagent-delivery 完成 openspec/changes/add-order-refund，主会话只保留关键结论。
```

安装后，Codex 在任务匹配 Skill 描述时也可能自动选择它。

## 执行流程

1. 锁定需求、验收标准、测试和发布边界。
2. 按独立验收阶段生成依赖 DAG；紧密耦合使用一个 Dev，长任务自动拆分并持续派发下一节点。
3. Dev 完成最小实现、条件性 TDD、focused tests、代码简化、自检和阶段 checkpoint。
4. 串行阶段或并行集成批次运行适用回归/compile/build；API 改动补真实 HTTP 参数测试。
5. 独立 Reviewer 审查固定范围；问题交回唯一 writer 修复、重新 checkpoint 并复查，直到通过或时限收口。

例如阶段 C/D 即使已有 checkpoint，只要共享文件、契约未冻结、数据库或 fixture 不能隔离，就必须串行。worktree 创建或集成条件失败时也自动回退串行并保留恢复证据。

## 示例

```text
请使用 $subagent-delivery 交付 Wayfinder 中已确认的 10 个开发阶段：
- 自动判断阶段依赖并持续启动下一任务；除非模块完全独立，否则串行；
- 每阶段自动创建本地 checkpoint commit，但不 push、不部署；
- 新增接口使用测试账号和实际订单参数调用本地 HTTP 路由；
- 最终独立 Review 并闭环 P0-P3；无法在时限内完成时列出未完成清单。
```

完整规则见 [SKILL.md](./SKILL.md)；恢复协议见 [checkpoint-and-recovery.md](./references/checkpoint-and-recovery.md)，模型策略见 [model-routing.md](./references/model-routing.md)，简报与测试证据见 [evidence-and-briefs.md](./references/evidence-and-briefs.md)。
