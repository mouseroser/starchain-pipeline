---
name: starchain
description: 星链（StarChain）— OpenClaw 多 agent 协作锻造流水线 v1.8。从 Step 1 分级到 Step 7 交付的全自动编排，覆盖 spec-kit 门控、交叉审查、修复循环、测试回归和 Epoch 回退。
---

# 星链 StarChain

星链是 OpenClaw 的多 agent 协作流水线，以 main（小光）为编排中心，链式串联所有 agent 完成从需求到交付的全流程。

使用本 skill 执行开发和修复任务，遵循 `references/pipeline-v1-8-contract.md` 中的流水线合约。

## When This Skill Triggers

Trigger this skill when the user asks to:
- run the cross-review pipeline
- assign work across review/coding/test/docs/monitor/brainstorming
- enforce Step 1.5 spec-kit gates
- handle Step 3/4/5 verdict loops and Epoch fallback
- create specs/plans/tasks for a feature (spec-kit workflow)

## Required Read

Before execution, read `references/pipeline-v1-8-contract.md`.

## Architecture: main 编排模式

main（小光）是顶层编排中心。所有 agent 由 main 直接 spawn（mode=run），main 在持久会话中串联全流程。

```
main（小光，读取本 SKILL.md 后充当编排中心）
├── Step 1：main 自己做分级 + 类型分析
├── Step 1.5：spawn gemini → NotebookLM 查询 → spawn brainstorming
├── Step 2：spawn coding（含 Step 2.5 冒烟）
├── Step 3：spawn review（只做交叉审查）
│   └── main 直接 grep 验证修复（不信 coding announce）
├── Step 4：spawn brainstorming → spawn coding → spawn review（循环，max 3 rounds）
├── Step 5：spawn test
├── Step 5.5：spawn brainstorming → spawn coding → spawn test（Epoch 循环，max 3）
├── Step 6：spawn gemini → spawn docs
├── Step 7：main 汇总交付 + message 通知晨星
└── 全程：main 补发关键推送到各职能群 + 监控群（sub-agent 推送不可靠）
```

### 知识层集成（NotebookLM）
- Step 1.5：查询 openclaw-docs / memory notebook 获取相关上下文
- Step 6：查询 notebook 辅助文档生成
- 所有 agent 可通过 nlm-gateway.sh 查询知识（按 ACL 权限）
- Notebook：memory / openclaw-docs / media-research

## Agent Roles

| Agent | Role | Key Steps |
|-------|------|-----------|
| main（小光） | 顶层编排中心 | Step 1, 1.5 编排, 4 编排, 5.5 编排, 7 |
| review | 交叉审查 | Step 3（结构化审查 + verdict） |
| coding | 开发执行 | Step 2, 2.5, 4(fix), TF(fix) |
| test | 测试执行 | Step 5, TF(rerun), Epoch(test) |
| brainstorming | 方案智囊 | Step 1.5(spec-kit), 4(方案), TF-2/3(方案), 5.5(分析) |
| docs | 文档生成 | Step 6 |
| gemini | 研究/文案加速 | Step 1.5(研究辅助), Step 6(润色), Step 5.5(诊断memo) |
| monitor-bot | 全局监控 | 全程状态 + 告警 |

## Spawn 规范

所有 agent 一律用 `mode=run` spawn：

```
sessions_spawn(agentId: "<agent>", mode: "run", task: "<任务+上下文>")
```

- main 是持久会话，每个 agent announce 回来后 main 继续下一步
- 不需要 mode=session，不需要 thread
- runTimeoutSeconds 按需设置（coding/brainstorming 建议 300s，其他默认）

### Spawn 重试机制（硬性要求）

任何 agent spawn 失败时（包括 LLM request timed out、503、网络错误等），main 必须自动重试：

1. 第一次失败 → 立即重试（相同参数）
2. 第二次失败 → 等待 10 秒后重试
3. 第三次仍失败 → 推送告警到监控群(-5131273722) + 通知晨星(target:1099011886)，标记该步骤为 BLOCKED

**绝不因为单次 spawn 失败就跳过步骤或 HALT。** 瞬时超时和 API 抖动是常见现象，重试通常能解决。

```
# 重试伪代码
for attempt in 1, 2, 3:
    result = sessions_spawn(...)
    if result.ok: break
    if attempt == 2: sleep(10)
    if attempt == 3: alert + BLOCKED
```

## Spec-Kit Integration (Step 1.5)

L2/L3 tasks must pass the Spec-Kit gate before Step 2. The workflow:
1. `specify` → specs/{feature}/spec.md (WHAT/WHY, no HOW)
2. `plan` → specs/{feature}/plan.md + research.md (tech decisions with rationale)
3. `tasks` → specs/{feature}/tasks.md (ordered, dependencies marked, [P] for parallel)
4. `analyze` → consistency check (spec ↔ plan ↔ tasks)

Critical consistency issues block Step 2. Brainstorming agent executes; main orchestrates.

### Optional Gemini Accelerator (Default: enabled for L2/L3)
- Step 1.5: main spawns gemini to generate clarification questions + risks + research bullets; feed into brainstorming context and fold into research.md.
- Step 6: before spawning docs, main spawns gemini to draft release-notes/FAQ outline; include as docs input.
- Step 5.5: when entering Epoch/HALT, main spawns gemini to summarize failure logs into a diagnosis memo for monitor-bot + delivery notes.

Reference: https://github.com/github/spec-kit

## Main 编排流程（逐步执行）

### Step 1：分级 + 类型分析
main 自己执行：
1. 判断等级：L1 / L2 / L3
2. 判断类型：A / B / C → 确定各 agent 模型配置
3. 推送分配单到监控群(-5131273722)
4. L1 → 快速通道 | L2/L3 → Step 1.5

### Step 1.5：Spec-Kit 门控（L2/L3）
main 编排：
1. spawn gemini → 产出澄清问题 + 风险 + 研究线索
2. spawn brainstorming → 产出 spec.md / plan.md / tasks.md / research.md
3. main 验证四件套一致性（Critical 问题阻塞 Step 2）
4. 推送结果到头脑风暴群(-5231604684) + 监控群

### Step 2 + 2.5：开发 + 冒烟
main 编排：
1. spawn coding → 按 tasks.md 开发 + 自执行冒烟测试
2. coding announce 回来后，main 检查结果
3. 冒烟 PASS → 创建 S1 快照，进入 Step 3
4. 冒烟 FAIL（coding 自修复 2 次仍 FAIL）→ 进入 Step 4

### Step 3：交叉审查
main 编排：
1. spawn review → 传入代码 diff + 需求，执行异模型交叉审查
2. review announce 回来后，main 解析 verdict JSON
3. PASS → 创建 S2 快照，进入 Step 5
4. PASS_WITH_NOTES → minor fix diff ≤ G2 免审 → S2 → Step 5；diff > G2 降级 NEEDS_FIX
5. NEEDS_FIX → 进入 Step 4

### Step 4：修复循环（max 3 rounds）
main 编排每轮：
1. 组装 context bundle（需求 + diff + issues JSON + 前轮反馈）
2. spawn brainstorming → 出修复方案
3. spawn coding → 执行修复
4. 回到 Step 3（spawn review 审查）
- R1: brainstorming sonnet/medium + coding codex/medium
- R2: brainstorming sonnet/medium + coding codex/medium
- R3: brainstorming opus/high + coding codex/xhigh
- R3 仍 NEEDS_FIX → Step 5.5

### Step 5：全量测试
main 编排：
1. spawn test → 执行完整测试
2. PASS → 创建 S3 快照，进入 Step 6
3. FAIL → 进入 TF 路径

### TF（Test Failure）路径
main 编排：
- TF-1: spawn coding 修复 → spawn test 重跑
- TF-2: spawn brainstorming(sonnet/medium) → spawn coding → spawn test
- TF-3: spawn brainstorming(opus/high) → spawn coding → spawn test + 全量回归
- diff > G3 → 回 Step 3（ReEntry++）
- ReEntry > 2 → 强制 Step 5.5

### Step 5.5：Epoch 回退（max 3 Epochs）
main 编排：
1. （可选）spawn gemini 产出诊断 memo
2. spawn brainstorming(opus/high) 分析根因，决定回滚到 S1/S2/继续
3. 回滚代码到选定快照
4. 重新走 Step 2 → 2.5 → 3 → 5
5. Epoch > 3 → HALT

### Step 6：文档（L1 跳过）
main 编排：
1. spawn gemini → 产出交付说明/FAQ 大纲
2. spawn docs → 生成/更新文档
3. 推送文档群(-5095976145) + 监控群

### Step 7：交付
main 自己执行：
1. 汇总交付摘要（scope + files + test + review + level + status + snapshots）
2. 推送监控群(-5131273722)
3. 通知晨星(target:1099011886)

## Execution Rules

0. **全自动推进，不停顿。** 除 Step 7 晨星确认外，所有步骤自动衔接，绝不暂停等待用户确认。进度推送到监控群即可，不要问"要继续吗"。
1. Always start at Step 1 (L1/L2/L3 classification + track selection).
2. L1 goes to fast lane; L2/L3 must pass Step 1.5 before Step 2.
3. main is orchestration center; all agents are direct executors.
4. Use structured verdict flow (`PASS`, `PASS_WITH_NOTES`, `NEEDS_FIX`) exactly as defined.
5. Apply weighted diff gates exactly:
   - G2 = 20 weighted lines
   - G3 = 30 weighted lines
6. Enforce bounded loops:
   - `ReEntry_MAX = 2`
   - Step 4 max 3 rounds
   - Step 5.5 max 3 Epochs
7. Use snapshots and timeout fallback exactly:
   - S1 after Step 2
   - S2 after Step 3 PASS
   - S3 after Step 5 PASS
   - only halt point: Epoch > 3
   - halt timeout fallback: 30min degraded delivery from S2
8. Step 7 delivery must include level, delivery status, and snapshot tags.
9. Step 1.5 spec-kit artifacts must be complete and consistent before Step 2.
10. Context bundle must be assembled and passed for every repair iteration.

## L1 快速通道

L1 跳过：Step 1.5 / Step 6
L1 流程：Step 1 → Step 2 → Step 2.5 → Step 3 → Step 5 → Step 7
- Step 3 额外输出 `"upgrade": null | "L2"`
- upgrade = "L2" → 从 Step 1.5 重新开始

## HALT 处理

Epoch > 3 或任何步骤超过 30 分钟无响应：
1. 推送告警到监控群(-5131273722)
2. 通知晨星(target:1099011886)
3. 以 degraded 状态交付当前最佳快照（S3 > S2 > S1）
4. Step 7 交付状态标记为 `degraded`

## 推送规范

main 在以下节点推送到监控群(-5131273722)：
- Step 1 分配单
- Step 1.5 Spec-Kit 结果
- Step 2 开发完成
- Step 2.5 冒烟结果
- Step 3 审查结论
- Step 4 每轮结果
- Step 5 测试结果
- Step 5.5 Epoch 状态
- Step 7 交付摘要 + 通知晨星

各 agent 也会默认推送到自己的职能群 + 监控群（见各 agent AGENTS.md）。

## Do Not Simplify Away Safety Nets

Do not skip:
- Step 1.5 spec-kit gate (L2/L3)
- Step 2.5 smoke gate
- Step 3 structured cross-review
- PASS_WITH_NOTES downgrade guards
- TF return-to-review rules
- Step 5.5 restart validation gate
- HALT 30min timeout fallback

If project context conflicts with the v1.8 contract, follow the flowchart contract and report the mismatch in final delivery notes.
