# 🔄 Pipeline Flowchart v2.1 — Constitution-First Multi-Agent Delivery with Gemini Fast Review

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
  🗂️ 协作材料规则
═══════════════════════════════════════════════════════════

  - 多 agent 联合工作的非正式产物
    （外部 GitHub 项目、本地镜像、共享分析素材、临时脚本等）
    → 统一放到 `intel/collaboration/`
  - 正式产物
    （报告、审查结论、修复代码、文档交付等）
    → 放各自 agent 目录
  - `intel/` 继续作为协作层，遵守单写者原则

═══════════════════════════════════════════════════════════
  📥 Step 1 — Task Classification & Lane Routing
═══════════════════════════════════════════════════════════

  ☀️ main（小光）/model opus /think high

  1. 📋 Classify level → L1 / L2 / L3
  2. 📋 Classify type →
       A 业务/架构 → coding(sonnet/medium) + review(gpt/high) + test(gpt/medium)
       B 算法/性能 → coding(gpt/medium) + review(gemini/medium) + test(sonnet/medium)
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
  🧭 Step 1.5 — Constitution-First Preflight (L2/L3)
  Orchestrator: ☀️ main
═══════════════════════════════════════════════════════════

  核心思想：
    先用 Gemini 做颗粒度对齐，把"问题定义"拉到同一层次
    再由 Claude（小克）基于问题宪法产出实施计划
    再由 Review/Gemini 复核一致性
    仅在高风险 / 明显分歧 / L3 场景下启用 Review/GPT 作为仲裁位

  ┌─────────────────────────────────────────────────────┐
  │  L1: 跳过重前置链，直接走快速通道                   │
  │  L2: Gemini → Claude → Review/Gemini               │
  │  L3: Gemini → Claude → Review/Gemini → Review/GPT  │
  └─────────────────────────────────────────────────────┘

  L2 标准链：
    1️⃣  spawn 织梦 (gemini) /think medium
        → 颗粒度对齐 / 问题定义同步
        → 产出「问题宪法」
        → 织梦群 (-5264626153) + 监控群

    2️⃣  spawn 小克 (claude) /think medium
        → 基于「问题宪法」产出实施计划
        → 结构：目标 / 边界 / 风险 / 执行步骤 / 依赖
        → 小克群 (-5101947063) + 监控群

    3️⃣  spawn review (swap 到 gemini) /think medium
        → 复核 claude 计划是否偏离原问题宪法
        → model: "gemini/gemini-3.1-pro-preview"
        → 输出：ALIGN / DRIFT / MAJOR_DRIFT
        → 交叉审核群 (-5242448266) + 监控群

       ALIGN ✅       → Step 2
       DRIFT ⚠️      Claude 修订 1 次 → Review/Gemini 再复核
       MAJOR_DRIFT ❌  → 升级 Review/GPT 仲裁位

  L3 升级链：
    1️⃣  spawn 织梦 (gemini) /think medium
        → 颗粒度对齐 / 问题宪法
        → 织梦群 + 监控群

    2️⃣  spawn 小克 (claude) /think medium
        → 基于问题宪法出实施计划
        → 小克群 + 监控群

    3️⃣  spawn review (swap 到 gemini) /think medium
        → 一致性复核
        → 交叉审核群 + 监控群

    4️⃣  spawn review (swap 到 gpt) /think high
        → 风险挑刺 / 反方审查 / 仲裁
        → model: "openai/gpt-5.4"
        → 输出：GO / REVISE / BLOCK
        → 交叉审核群 + 监控群

       GO ✅      → Step 2
       REVISE ⚠️  → Claude 修订 1 次 → Review/Gemini 复核 → 再判定
       BLOCK ❌   → main 推送阻塞原因 → HALT / 降级交付

  规则：
    - 🚫 不新增晨星中途确认节点，全自动推进
    - 🚫 这条前置链不替换后续开发 / 审查 / 测试主干
    - ✅ 一致性复核是 review 的职责，通过 swap 模型实现
    - ✅ gemini 只负责问题宪法，不直接做复核
    - ✅ 这是 v2.1 对 Step 1.5 的优化，融入 claude agent + gemini 快速复核

═══════════════════════════════════════════════════════════
  ⚡ L1 快速通道
═══════════════════════════════════════════════════════════

  💻 coding（默认配置）→ 跳过重前置链
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
  按 Step 1.5 最终计划顺序执行开发
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
  🔍 Step 3 — Cross-Review + Gemini Fast Check
  Executor: 🔍 review
  Orchestrator: ☀️ main
═══════════════════════════════════════════════════════════

  三模型交叉审核思路：
    - Claude / GPT 作为主交叉审查模型对
    - Gemini 参与快速复核与算法分析
    - 高分歧时再进入仲裁位

  main 根据 Step 1 判定的 Type 动态 spawn：

  Type A（业务/架构）:
    1️⃣ spawn review (swap gpt/high)
       → 结构化交叉审查
       → 交叉审核群 + 监控群

    2️⃣ spawn review (swap gemini/medium)
       → 快速复核代码是否符合需求
       → 交叉审核群 + 监控群

    综合判定：
      两者都 PASS ✅ → 📸 S2 → Step 5
      任一 NEEDS_FIX ❌ → Step 4

  Type B（算法/性能）:
    1️⃣ spawn review (swap gemini/medium)
       → 算法分析 + 性能审查
       → 交叉审核群 + 监控群

    判定：
      PASS ✅              → 📸 S2 → Step 5
      PASS_WITH_NOTES ⚠️   → minor fix → diff ≤ G2 免审 → 📸 S2 → Step 5
                                      → diff > G2 降级 NEEDS_FIX → Step 4
      NEEDS_FIX ❌         → Step 4

  优化收益：
    • Type A: gemini 快速复核可提前发现 30-40% 的需求偏离问题
    • Type B: gemini 擅长算法分析，比 sonnet 更适合

  分歧仲裁（reviewer 提出问题 + coder 反驳时触发）：
    → GPT / Claude 主审之外，必要时追加 Gemini 诊断摘要
    → review(opus/medium) + coding(gpt/xhigh)
    → 如果 coder 不反驳，reviewer 判定直接生效，无需仲裁

  输出结构化 JSON：
  {
    "verdict": "PASS | PASS_WITH_NOTES | NEEDS_FIX",
    "issues": [{ severity, category, file, line, description, suggestion }]
  }

  🔍 → 审核群 + 监控群

═══════════════════════════════════════════════════════════
  🔧 Step 4 — Repair Loop + Gemini Pre-Review (max 3 rounds)
  Orchestrator: ☀️ main
  方案: 🧠 brainstorming │ 执行: 💻 coding │ 预审: 🔍 review/gemini
═══════════════════════════════════════════════════════════

  每轮必须携带 context bundle：
  ┌─────────────────────────────────────────────────────┐
  │ - 原始需求                                          │
  │ - Step 1.5 最终计划 / 问题宪法                       │
  │ - 当前代码 diff                                     │
  │ - 审查结构化反馈（issues JSON）                      │
  │ - 前轮修复 diff + 前轮审查反馈（如有）                │
  │ - 冒烟测试失败日志（如有）                            │
  └─────────────────────────────────────────────────────┘

  每轮流程：
    1. spawn brainstorming → 出修复方案
    2. spawn coding → 执行修复
    3. spawn review (swap gemini/low) → 快速预审
       预审 PASS ✅ → 进入正式审查（Step 3）
       预审 FAIL ❌ → 直接进入下一轮修复（节省成本）

  🔁 R1: brainstorming sonnet/low + coding gpt/medium + review/gemini 预审
  🔁 R2: brainstorming sonnet/medium + coding gpt/medium + review/gemini 预审
  🔁 R3: brainstorming opus/high + coding gpt/xhigh + review/gemini 预审

       ✅ Step 3 PASS / PASS_WITH_NOTES → Step 5
       ❌ R3 still NEEDS_FIX → Step 5.5 (Epoch Fallback)

  优化收益：
    • gemini 预审可提前发现 30-40% 的明显问题
    • 避免浪费 gpt/sonnet 的高成本审查
    • 预计降本 15-20%

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
     → 织梦群 + 监控群

  2. spawn brainstorming /model opus /think high
     → 根因分析 + 回滚决策
     → 灵感熔炉群 + 监控群

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
  Executor: 📝 docs
  Orchestrator: ☀️ main
═══════════════════════════════════════════════════════════

  1. spawn notebooklm /think medium
     → 查询文档模板 / 历史知识
     → 珊瑚群 + 监控群

  2. spawn gemini /think medium
     → 产出交付说明 / FAQ 大纲
     → 织梦群 + 监控群

  3. spawn docs
     → 生成/更新文档
     → 文档群 + 监控群

  输入：最终 diff + 需求 + 审查摘要 + Step 1.5 产出

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
    🧭 Constitution / Plan summary
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
│  Step 1.5: gemini → claude → review/gemini             │
│            (+ review/gpt 仲裁 for L3 / major drift)    │
│  Step 2:   coding → sonnet/medium                      │
│  Step 3:   review → gpt/high + gemini/medium (快速复核) │
│  Step 4:   review/gemini 预审 (low)                    │
│  Step 5:   test → gpt/medium                           │
│  仲裁:     review(opus/medium) + coding(gpt/xhigh)     │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  Type B (算法/性能)                                      │
├─────────────────────────────────────────────────────────┤
│  Step 1.5: gemini → claude → review/gemini             │
│            (+ review/gpt 仲裁 for L3 / major drift)    │
│  Step 2:   coding → gpt/medium                         │
│  Step 3:   review → gemini/medium (算法分析)            │
│  Step 4:   review/gemini 预审 (low)                    │
│  Step 5:   test → sonnet/medium                        │
│  仲裁:     review(opus/medium) + coding(gpt/xhigh)     │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  修复升级路径（所有 Type）                               │
├─────────────────────────────────────────────────────────┤
│  Step 4 R1/R2: brainstorming(sonnet/medium)            │
│                 + coding(gpt/medium)                   │
│                 + review/gemini 预审 (low)             │
│  Step 4 R3:     brainstorming(opus/high)               │
│                 + coding(gpt/xhigh)                    │
│                 + review/gemini 预审 (low)             │
│  TF-2:          brainstorming(sonnet/medium)           │
│  TF-3:          brainstorming(opus/high)               │
│  Step 5.5:      gemini(high) + brainstorming(opus/high)│
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  固定配置                                                │
├─────────────────────────────────────────────────────────┤
│  main:          opus/high                              │
│  claude (小克): 实施计划器 (Step 1.5.2)                │
│  review:        交叉审查 + 复核 + 仲裁 + 预审           │
│                 - Step 1.5.3: swap gemini (一致性复核)  │
│                 - Step 1.5.4: swap gpt (高风险仲裁)     │
│                 - Step 3: 根据 Type A/B 动态 swap       │
│                 - Step 4: swap gemini (快速预审)        │
│  gemini (织梦): 问题宪法 (Step 1.5.1) + 快速复核 + 预审 │
│  珊瑚(nlm):     知识查询 / 模板 / 历史知识              │
│  brainstorming: 方案生成（默认 sonnet，动态切换 opus）  │
└─────────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════
  🛡️ Tool Resilience & Degradation
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

  默认原则：
    • 有消息能力的 agent → 自己的职能群通知是主链路
    • main → 监控群、缺失补发、最终交付、告警

  通知粒度：
    • 简单委派：开始 / 完成 / 失败
    • 多智能体编排：开始 / 关键阶段 / 完成 / 失败
    • 正式流水线：按实际阶段逐段通知

  监控群 (-5131273722) 推送节点：
    • Step 1: 分配单
    • Step 1.5: Constitution / Plan / Review 结果
    • Step 2: 开发完成
    • Step 2.5: 冒烟结果
    • Step 3: 审查结论
    • Step 4: 每轮结果
    • Step 5: 测试结果
    • Step 5.5: Epoch 状态
    • Step 7: 交付摘要

  各 agent 职能群推送：
    • gemini / 织梦 → 织梦群 (-5264626153)
    • claude / 小克 → 小克群 (-5101947063)
    • review → 交叉审核群 (-5242448266)
    • coding → 代码编程群 (-5039283416)
    • test → 代码测试群 (-5245840611)
    • brainstorming / 灵感熔炉 → 灵感熔炉群 (-5231604684)
    • docs → 项目文档群 (-5095976145)
    • notebooklm / 珊瑚 → 珊瑚群 (-5202217379)

  ⚠️  同一问题若沿同一方向连续三次仍未解决，必须换方向

═══════════════════════════════════════════════════════════
```

## 🎯 核心变更（v2.0 → v2.1）

**增强 Gemini 快速复核与预审机制：**

1. **Step 3 Type A 增加 gemini 快速复核**
   - gpt 结构化审查 + gemini 快速复核需求符合度
   - 提前发现 30-40% 的需求偏离问题

2. **Step 3 Type B 改用 gemini**
   - 从 sonnet → gemini
   - gemini 擅长算法分析，更适合 Type B

3. **Step 4 增加 gemini 预审**
   - 每轮修复后先用 gemini 快速预审
   - 预审 PASS 才进入正式审查
   - 预审 FAIL 直接下一轮，节省成本
   - 预计降本 15-20%

**优化收益：**
- ✅ 提升效率 30-40%
- ✅ 降低成本 15-20%
- ✅ 充分利用 gemini 快速分析能力
