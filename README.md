# 星链 StarChain

OpenClaw 多 agent 协作流水线 v1.8。以 main（小光）为编排中心，链式串联所有 agent 完成从需求到交付的全流程。

## 流程概览

```
Step 1    分级（L1/L2/L3）+ 类型分析（A/B/C）
Step 1.5  gemini 研究 → NotebookLM 查询 → brainstorming spec-kit 四件套（L2/L3）
Step 2    coding 开发 + Step 2.5 冒烟测试
Step 3    review 交叉审查（PASS / PASS_WITH_NOTES / NEEDS_FIX）
Step 4    修复循环：brainstorming 方案 → coding 修复 → review 审查（max 3 rounds）
Step 5    test 全量测试
  TF      测试失败路径（TF-1/2/3）
Step 5.5  Epoch 回退（max 3 Epochs）
Step 6    gemini 大纲 → docs 文档生成
Step 7    交付汇总 → 通知晨星
```

## Agent 角色

| Agent | 职责 | 关键步骤 |
|-------|------|----------|
| main（小光） | 编排中心 | Step 1, 1.5 编排, 4 编排, 7 |
| review | 交叉审查 | Step 3 |
| coding | 开发执行 | Step 2, 4(fix), TF(fix) |
| test | 测试执行 | Step 5, TF(rerun) |
| brainstorming | 方案智囊 | Step 1.5(spec-kit), 4(方案) |
| docs | 文档生成 | Step 6 |
| gemini | 研究/文案加速 | Step 1.5(研究), 6(润色) |
| monitor-bot | 全局监控 | 全程状态 + 告警 |

## 核心机制

- **Spec-Kit 门控**：L2/L3 必须通过四件套（spec/plan/tasks/research）一致性检查
- **Spawn 重试**：失败自动重试 3 次（第 2 次等 10 秒），3 次失败才 BLOCKED
- **快照系统**：S1(开发后) → S2(审查后) → S3(测试后)，支持回滚
- **有界循环**：Step 4 max 3 rounds，Epoch max 3，ReEntry max 2
- **全自动推进**：除 Step 7 晨星确认外，所有步骤自动衔接，不停顿
- **NotebookLM 集成**：Step 1.5 知识查询，Step 6 辅助文档生成

## 分级说明

| 等级 | 说明 | 流程 |
|------|------|------|
| L1 | 简单修复/小功能 | 1→2→2.5→3→5→7（跳过 1.5/6） |
| L2 | 标准功能开发 | 1→1.5→2→2.5→3→5→6→7 |
| L3 | 复杂系统/架构变更 | 完整流程 + 更严格的审查标准 |

## 推送规范

所有 agent 在关键节点推送到各自职能群 + 监控群(-5131273722)。main 补发关键推送（sub-agent 推送不可靠）。

## HALT 处理

Epoch > 3 或超时 30 分钟 → 以 degraded 状态交付当前最佳快照 → 通知晨星。

## 依赖

- OpenClaw agent 生态（review / coding / test / docs / brainstorming / gemini / monitor-bot）
- [NotebookLM Skill](~/.openclaw/skills/notebooklm/) 知识层
- [自媒体流水线](https://github.com/mouseroser/wemedia-pipeline)（内容创作场景）

## 许可

MIT
