# 🔄 Pipeline Flowchart v1.8 — Cross-Review Multi-Agent Delivery

```
┌─────────────────────────────────────────────────────────┐
│  🌐 Global State                                        │
│  ReEntry = 0 │ ReEntry_MAX = 2                          │
│  Snapshots: S1, S2, S3                                  │
│  HALT_TIMEOUT = 30min                                   │
│  G2 = 20 加权行 │ G3 = 30 加权行                        │
│  Weight: critical paths ×2 │ others ×1                  │
└─────────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════
  📥 Step 1 — Task Classification & Lane Routing
═══════════════════════════════════════════════════════════

  ☀️ main（小光）/model opus /think high

  1. 📋 Classify level → L1 / L2 / L3
  2. 📋 Classify type →
       A 业务/架构 → coding(sonnet/medium) + review(gpt/high) + test(gpt/medium)
       B 算法/性能 → coding(gpt/medium) + review(sonnet/medium) + test(sonnet/medium)
       C 混合     → 拆分走 A/B
  3. 输出结构化分配单 → 📋 监控群

       ┌──── L1 ────┐
       │  ⚡ Fast    │
       │   Lane     │──────────────────────────► L1 快速通道
       └────────────┘

       ┌── L2/L3 ──┐
       │ 🛤️ Standard│
       │   Lane     │──────────────────────────► Step 1.5
       └────────────┘

═══════════════════════════════════════════════════════════
  📐 Step 1.5 — Spec-Kit Gate (L2/L3 Only)
  Based on: github.com/github/spec-kit
  Executor: 🧠 brainstorming │ Orchestrator: ☀️ main
═══════════════════════════════════════════════════════════

  打磨层流程（织梦打磨 + Opus 验证）：
    0️⃣  spawn 织梦 (gemini) /think medium
        → 产出澄清问题 + 风险 + 研究线索 + 初稿草案

    0️⃣.5 spawn review /model opus
        → 验证织梦产出，优化逻辑

  四步流程：
    1️⃣  specify → specs/{feature}/spec.md
        ✅ 聚焦 WHAT/WHY，不涉及 HOW
        ⚠️  模糊点标记 [NEEDS CLARIFICATION]

    2️⃣  plan → specs/{feature}/plan.md
              + specs/{feature}/research.md
              + specs/{feature}/contracts/（如需要）
              + specs/{feature}/data-model.md（如需要）
        ✅ 技术选型必须有理由
        ✅ 架构决策追溯到 spec 需求

    3️⃣  tasks → specs/{feature}/tasks.md
        ✅ 按用户故事分组
        ✅ 标注依赖关系，[P] 标记可并行
        ✅ 每个任务包含文件路径

    4️⃣  analyze → 一致性检查（spec ↔ plan ↔ tasks）
        🚫 Critical → 修复后重新检查
        ⚠️  Warning → 建议处理
        ℹ️  Info → 优化建议

       ✅ Gate passed → Step 2
       📐 → 头脑风暴群 + 监控群

═══════════════════════════════════════════════════════════
  ⚡ L1 快速通道
═══════════════════════════════════════════════════════════

  💻 coding（默认配置）→ 跳过 Spec-Kit + brainstorming
       │
       ▼
  🧪 冒烟测试
       PASS ✅ → 单轮交叉审查
       FAIL ❌ → 修复 1 次 → 仍 FAIL → 升级 L2（从 Step 1.5）
       │
       ▼
  🔍 单轮交叉审查（结构化 JSON）
       额外字段："upgrade": null | "L2"
       upgrade: "L2" → 升级，从 Step 1.5 开始
       PASS ✅ → 跳过文档 → Step 7
       NEEDS_FIX ❌ → 修复 1 次 → 仍不过 → 升级 L2

  L1 全程无挂起：走不通自动升级 L2

═══════════════════════════════════════════════════════════
  🔨 Step 2 — Development
  Executor: 💻 coding
  Orchestrator: ☀️ main
═══════════════════════════════════════════════════════════

  main 根据 Step 1 判定的 Type 动态 spawn：

       ▼                    ▼
  ┌──────────────┐   ┌──────────────┐
  │ 💻 coding   │   │ 💻 coding   │
  │ (sonnet/med)│   │ (gpt/med)   │
  │  (Type A)   │   │  (Type B)   │
  └──────┬───────┘   └──────┬───────┘
         └───────┬───────────┘
                 ▼
  按 tasks.md 顺序执行开发
  💻 → 编程群 + 监控群
                 │
                 ▼
          📸 Create S1 snapshot

═══════════════════════════════════════════════════════════
  🔥 Step 2.5 — Smoke Test Gate
  Executor: 💻 coding（自执行）
═══════════════════════════════════════════════════════════

  检查项：编译/语法 ✓ │ 单测 ✓ │ 基本功能 ✓

       ❌ Fail → 自修复（max 2 次）→ 仍 FAIL → Step 4（带日志）
       ✅ Pass → Step 3
  🧪 → 测试群 + 监控群

═══════════════════════════════════════════════════════════
  🔍 Step 3 — Structured Cross-Review
  Executor: 🔍 review
  Orchestrator: ☀️ main
═══════════════════════════════════════════════════════════

  main 根据 Step 1 判定的 Type 动态 spawn：

  Type A: coding(sonnet) 代码 → review(gpt/high) 交叉审查
  Type B: coding(gpt) 代码 → review(sonnet/medium) 交叉审查

  分歧仲裁（reviewer 提出问题 + coder 反驳时触发）：
    → review(opus/medium) + coding(gpt/xhigh)
    → 如果 coder 不反驳，reviewer 判定直接生效，无需仲裁

  输出结构化 JSON：
  {
    "verdict": "PASS | PASS_WITH_NOTES | NEEDS_FIX",
    "issues": [{ severity, category, file, line, description, suggestion }]
  }

  判定：
    0 crit + 0 major + 0 minor   → PASS
    0 crit + 0 major + minor ≤ 3 → PASS_WITH_NOTES
    存在 crit 或 major           → NEEDS_FIX

  ┌──────────────────────┐
  │ ✅ PASS              │──► 📸 S2 → Step 5
  │ ⚠️  PASS_WITH_NOTES  │──► minor fix → diff ≤ G2 免审 → 📸 S2 → Step 5
  │                      │            → diff > G2 降级 NEEDS_FIX → Step 4
  │ ❌ NEEDS_FIX         │──► Step 4
  └──────────────────────┘
  🔍 → 审核群 + 监控群

═══════════════════════════════════════════════════════════
  🔧 Step 4 — Repair Loop (max 3 rounds)
  Orchestrator: ☀️ main
  方案: 🧠 brainstorming │ 执行: 💻 coding
═══════════════════════════════════════════════════════════

  每轮必须携带 context bundle：
  ┌─────────────────────────────────────────────────────┐
  │ - 原始需求                                          │
  │ - 当前代码 diff                                     │
  │ - 审查结构化反馈（issues JSON）                      │
  │ - 前轮修复 diff + 前轮审查反馈（如有）                │
  │ - 冒烟测试失败日志（如有）                            │
  └─────────────────────────────────────────────────────┘

  🔁 R1: brainstorming sonnet/medium + coding gpt/medium → Step 3
  🔁 R2: brainstorming sonnet/medium + coding gpt/medium → Step 3
  🔁 R3: brainstorming opus/high + coding gpt/xhigh → Step 3

       ✅ Step 3 PASS / PASS_WITH_NOTES → Step 5
       ❌ R3 still NEEDS_FIX → Step 5.5 (Epoch Fallback)

═══════════════════════════════════════════════════════════
  🧪 Step 5 — Test Execution
  Executor: 🧪 test
  Orchestrator: ☀️ main
═══════════════════════════════════════════════════════════

  main 根据 Step 1 判定的 Type 动态 spawn：

       ▼                    ▼
  ┌──────────────┐   ┌──────────────┐
  │ 🧪 test     │   │ 🧪 test     │
  │ (gpt/med)   │   │ (sonnet/med)│
  │  (Type A)   │   │  (Type B)   │
  └──────┬───────┘   └──────┬───────┘
         └───────┬───────────┘
                 ▼
  🧪 → 测试群 + 监控群

       ✅ Pass → 📸 Create S3 snapshot → Step 6
       ❌ Fail → 🔥 TF Path

  ┌─────────────────────────────────────────────────────┐
  │  🔥 TF Recovery Path                                │
  │  输入：失败日志 + 代码 + 需求 + context bundle       │
  │                                                     │
  │  TF-1: coding 直接修复 + test 重跑                   │
  │  TF-2: brainstorming sonnet/medium + coding + test   │
  │  TF-3: brainstorming opus/high + coding + test      │
  │         + 全量回归                                   │
  │                                                     │
  │  📏 TF-1/2 PASS 后检查 diff:                        │
  │     diff ≤ G3 → 全量回归 → PASS → Step 6            │
  │     diff > G3 → Step 3 (ReEntry++)                  │
  │     ReEntry > 2 → 强制 Step 5.5                     │
  │                                                     │
  │  📏 TF-3: 必须回 Step 3 (ReEntry++)                 │
  │     ReEntry > 2 → 强制 Step 5.5                     │
  │                                                     │
  │  ❌ TF-3 fail → Step 5.5                            │
  └─────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════
  ♻️  Step 5.5 — Epoch Fallback (max 3 Epochs)
  Orchestrator: ☀️ main
═══════════════════════════════════════════════════════════

  Entry: Step4-R3 fail OR TF-3 fail

  1. spawn gemini /think high
     → 产出诊断 memo

  2. spawn brainstorming /model opus /think high
     → 根因分析 + 回滚决策

  回滚选项（每 Epoch 开始时选择）：
    🔙 Rollback to S1
    🔙 Rollback to S2
    ▶️  Continue without rollback

  Epoch N:
    R1: brainstorming sonnet/medium → coding → 🧪 增量测试
        PASS → Epoch 结束 │ FAIL → R2
    R2: brainstorming sonnet/medium → coding → 🧪 增量测试
        PASS/FAIL → Epoch 结束

    Epoch 结束 → 🧪 全量回归
      PASS ✅ → Step 6
      FAIL ❌ → Epoch ≤ 3?
        是: 🧠 重启分析 opus/high → 🔍 校验 gpt/high
            → 通过: 新 Epoch(N+1)
            → 打回: 修正 1 次 → 新 Epoch(N+1)
        否: 🛑 HALT

  ┌─────────────────────────────────────────────────────┐
  │  🛑 HALT — 30min Timeout Fallback                   │
  │                                                     │
  │  ⏱️  Start 30-minute timer                          │
  │  On timeout:                                        │
  │    📋 GPT /think high 生成诊断报告                   │
  │    📦 从 S2 降级交付 (status: degraded)              │
  │    🔔 Notify monitor + 晨星                         │
  └─────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════
  📝 Step 6 — Documentation (skip for L1)
  Executor: 📝 docs │ /model opus /think high
  Orchestrator: ☀️ main
═══════════════════════════════════════════════════════════

  1. spawn gemini /think medium
     → 产出交付说明/FAQ 大纲

  2. spawn docs /model opus
     → 生成/更新文档

  输入：最终 diff + 需求 + 审查摘要 + spec-kit 产出
  📝 → 文档群 + 监控群

═══════════════════════════════════════════════════════════
  📦 Step 7 — Final Delivery
  Executor: ☀️ main /model opus /think high
═══════════════════════════════════════════════════════════

  交付摘要必须包含：
    ✅ Completed scope
    📁 Changed files
    🧪 Test results
    🔍 Review conclusion
    🏷️  Level: L1 / L2 / L3
    🏷️  Type: A / B / C
    📊 Delivery status: normal | degraded
    📸 Snapshot tags: S1, S2, S3

  → 监控群 (-5131273722)
  → 晨星 (target:1099011886) ✅

═══════════════════════════════════════════════════════════
  🏁 END
═══════════════════════════════════════════════════════════

═══════════════════════════════════════════════════════════
  📊 Model Configuration Matrix
═══════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────┐
│  Type A (业务/架构)                                      │
├─────────────────────────────────────────────────────────┤
│  Step 2:  coding → sonnet/medium                        │
│  Step 3:  review → gpt/high                             │
│  Step 5:  test → gpt/medium                             │
│  仲裁:    review(opus/medium) + coding(gpt/xhigh)       │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  Type B (算法/性能)                                      │
├─────────────────────────────────────────────────────────┤
│  Step 2:  coding → gpt/medium                           │
│  Step 3:  review → sonnet/medium                        │
│  Step 5:  test → sonnet/medium                          │
│  仲裁:    review(opus/medium) + coding(gpt/xhigh)       │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  修复升级路径（所有 Type）                               │
├─────────────────────────────────────────────────────────┤
│  Step 4 R1/R2: brainstorming(sonnet/medium)             │
│                + coding(gpt/medium)                     │
│  Step 4 R3:    brainstorming(opus/high)                 │
│                + coding(gpt/xhigh)                      │
│  TF-2:         brainstorming(sonnet/medium)             │
│  TF-3:         brainstorming(opus/high)                 │
│  Step 5.5:     brainstorming(opus/high)                 │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  固定配置                                                │
├─────────────────────────────────────────────────────────┤
│  main:         opus/high                                │
│  monitor-bot:  minimax                                  │
│  docs:         minimax/high                             │
│  织梦(gemini): gemini-preview (Step 1.5: medium,        │
│                                Step 5.5: high)          │
│  brainstorming: sonnet (默认，动态切换到 opus)          │
└─────────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════
  🛡️ Tool Resilience & Degradation (Step 3.5)
═══════════════════════════════════════════════════════════

  任何外部 CLI 依赖失败时（如 nlm-gateway.sh）：
    1. 发送 Warning 告警到监控群
    2. 跳过失败的工具调用，回退到模型权重
    3. 继续推进到下一步

  🚫 绝对禁止因工具失败中断流水线

═══════════════════════════════════════════════════════════
  🔄 Spawn Retry Mechanism
═══════════════════════════════════════════════════════════

  任何 agent spawn 失败时：
    1️⃣  第一次失败 → 立即重试（相同参数）
    2️⃣  第二次失败 → 等待 10 秒后重试
    3️⃣  第三次仍失败 → 推送告警到监控群 + 通知晨星
                      → 标记该步骤为 BLOCKED

  🚫 绝不因单次 spawn 失败就跳过步骤或 HALT

═══════════════════════════════════════════════════════════
  📢 Push Notification Rules
═══════════════════════════════════════════════════════════

  监控群 (-5131273722) 推送节点：
    • Step 1: 分配单
    • Step 1.5: Spec-Kit 结果
    • Step 2: 开发完成
    • Step 2.5: 冒烟结果
    • Step 3: 审查结论
    • Step 4: 每轮结果
    • Step 5: 测试结果
    • Step 5.5: Epoch 状态
    • Step 7: 交付摘要

  各 agent 职能群推送：
    • review → 交叉审核群 (-5242448266)
    • coding → 代码编程群 (-5039283416)
    • test → 代码测试群 (-5245840611)
    • brainstorming → 头脑风暴群 (-5231604684)
    • docs → 项目文档群 (-5095976145)
    • 织梦(gemini) → 织梦群 (-5264626153)

  ⚠️  sub-agent 推送不可靠，关键推送由 main 补发

═══════════════════════════════════════════════════════════
```
