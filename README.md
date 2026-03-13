# 星链 StarChain

OpenClaw 多 agent 协作流水线 v2.5。以 main（小光）为编排中心，按 `Constitution-First → NotebookLM 补料 → Spec-Kit 落地 → coding/test/docs` 的主链完成从需求到交付的全流程。

## 流程概览

```
Step 1    分级（L1/L2/L3）+ 类型分析（A/B/C）
Step 1.5  Constitution-First（gemini 扫描 → openai 宪法 → claude 计划 → review/gemini 复核 → review/openai|claude 仲裁）
Step 1.5S NotebookLM 补料 → brainstorming 落地 Spec-Kit 四件套（spec/plan/tasks/research）→ analyze 一致性检查（L2/L3 Step 2 gate）
Step 2    coding 开发 + Step 2.5 冒烟测试（默认按 tasks.md 执行）
Step 3    review 交叉审查（claude + gemini，按需 openai|claude 仲裁）
Step 4    修复循环：brainstorming 方案 → coding 修复 → review 预审/复审（max 3 rounds）
Step 5    test 全量测试
  TF      测试失败路径（TF-1/2/3）
Step 5.5  Epoch 回退 / 诊断 / 仲裁（max 3 Epochs）
Step 6    notebooklm 模板 → gemini 大纲 → docs 文档生成
Step 7    交付汇总 → 通知晨星
```

## Agent 角色

| Agent | 职责 | 关键步骤 |
|-------|------|----------|
| main（小光） | 编排中心 | Step 1, 1.5, 1.5S, 4, 5.5, 7 |
| openai | 宪法定稿 + 仲裁 | Step 1.5B, 1.5E, 3(分歧仲裁), 5.5 |
| claude | 主方案 / 主审查 | Step 1.5C, 3 |
| review | 交叉评审中枢 | Step 1.5D, 1.5E, 3, 4 |
| coding | 开发执行 | Step 2, 2.5, 4(fix), TF(fix) |
| test | 测试执行 | Step 5, TF(rerun), 5.5 |
| brainstorming | 方案智囊 + Spec-Kit 落地 | Step 1.5S(spec/plan/tasks/research), 4 |
| notebooklm | 知识补料 / 模板支撑 | Step 1.5S, 6 |
| gemini | 扫描 / 反方 review / 文档大纲 | Step 1.5A, 1.5D, 3, 5.5, 6 |
| docs | 文档生成 | Step 6 |
| monitor-bot | 全局监控 | 全程状态 + 告警 |

## 核心机制

- **Constitution-First**：先锁定歧义、边界、红线、计划与复核结论，再进入执行链
- **Spec-Kit 门控**：L2/L3 必须在 Step 1.5S 通过 `spec/plan/tasks/research` 一致性检查
- **NotebookLM 默认保留**：在 Step 1.5S 提供知识补料，在 Step 6 提供模板支撑
- **`tasks.md` 默认入口**：`coding` 默认按 `tasks.md` 执行，并同时遵守最终宪法与批准计划
- **Spawn 重试**：失败自动重试 3 次（第 2 次等 10 秒），3 次失败才 BLOCKED
- **快照系统**：S1(开发后) → S2(审查后) → S3(测试后)，支持回滚
- **有界循环**：Step 4 max 3 rounds，Epoch max 3，ReEntry max 2
- **全自动推进**：除 Step 7 晨星确认外，所有步骤自动衔接，不停顿

## 分级说明

| 等级 | 说明 | 流程 |
|------|------|------|
| L1 | 简单修复/小功能 | 1→2→2.5→3→5→7（跳过 1.5/1.5S/6） |
| L2 | 标准功能开发 | 1→1.5→1.5S→2→2.5→3→5→6→7 |
| L3 | 复杂系统/架构变更 | 完整流程 + 更严格的审查标准 |

## 推送规范

所有关键节点的消息统一由 main 推送到对应职能群 + 监控群(-5131273722)。sub-agent 自行推送仅视为 best-effort，不作为可靠通知链路。

## 最近更新（2026-03-13）

- 明确采用 **main-first** 通知模型
- sub-agent 自推降级为 **best-effort**
- 监控群可靠可见性统一由 `main` 保证
- 历史 contract 文档已同步对齐，避免旧规则造成误导

## HALT 处理

Epoch > 3 或超时 30 分钟 → 以 degraded 状态交付当前最佳快照 → 通知晨星。

## 依赖

- OpenClaw agent 生态（openai / claude / review / coding / test / docs / brainstorming / gemini / notebooklm / monitor-bot）
- [NotebookLM Skill](~/.openclaw/skills/notebooklm/) 知识层
- [自媒体流水线](https://github.com/mouseroser/wemedia-pipeline)（内容创作场景）

## 许可

MIT
