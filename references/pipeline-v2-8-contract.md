# 星链流水线 v2.8 合约

**版本**: v2.8  
**生效日期**: 2026-03-12  
**上一版本**: v2.7（回滚参考）

## 核心变更

### v2.8 主要改进（相比 v2.7）

1. **NotebookLM 提前介入**：从 Step 1.5S 移到 Step 1.5B，在制定宪法之前提供历史经验
2. **证据驱动规则**：宪法基于 NotebookLM 的历史教训制定，避免重复踩坑
3. **Brainstorming 动态模型**：根据任务级别和轮次动态选择 sonnet/opus
4. **与星鉴统一**：NotebookLM 深度参与模式与星鉴流水线一致

### v2.7 主要改进（相比 v2.6）

1. **Launcher Script 模式**：Main 不再在私聊里串联所有步骤，改用 `starchain-launcher.sh` 一键启动
2. **统一独立性规则**：与星鉴流水线统一仲裁独立性规则
3. **主私聊不占线**：符合"主私聊不承载长编排"约束

### 优化收益预期

- **效率提升**: 30-40%（减少 main 等待时间）
- **降本**: 35-45%（优化模型使用 + Brainstorming 动态分配）
- **成功率**: Epoch 成功率提升 15-20%
- **质量提升**: 宪法基于历史经验，避免重复踩坑
- **用户体验**: 主私聊不再"像不可用"

---

## 架构：Launcher Script 模式

```
main（小光）
└── exec ~/.openclaw/skills/starchain/scripts/starchain-launcher.sh "<任务>" <L1|L2|L3> [project-path]
    ├── Step 1：任务分级 + 类型分析（脚本记录）
    ├── Step 1.5：Constitution-First 打磨层（NotebookLM 深度参与）
    │   ├── openclaw agent --agent gemini → 扫描
    │   ├── openclaw agent --agent notebooklm → 深度研究（提前介入）
    │   ├── openclaw agent --agent openai → 宪法（基于历史经验）
    │   ├── openclaw agent --agent claude → 计划（基于最佳实践）
    │   ├── openclaw agent --agent gemini → 一致性复核
    │   ├── openclaw agent --agent openai/claude → 仲裁（按需）
    │   └── openclaw agent --agent brainstorming → Spec-Kit 落地
    ├── Step 2：openclaw agent --agent coding → 开发（含 Step 2.5 冒烟）
    ├── Step 3：并行 spawn claude + gemini → 双审
    ├── Step 4：循环修复（max 3 rounds）
    ├── Step 5：openclaw agent --agent test → 测试
    ├── Step 5.5：Epoch 诊断与仲裁（按需）
    ├── Step 6：openclaw agent --agent docs → 文档
    └── Step 7：main 汇总交付 + 通知晨星

**优势**：
- Main 不在私聊里等待和轮询
- Launcher 自动串联，无需手动编排
- 符合"主私聊不承载长编排"约束
- NotebookLM 深度参与，宪法基于历史经验
- 证据驱动规则，避免重复踩坑

### 产出目录规范

**所有星链流水线产物统一存放于 `intel/collaboration/starchain/`**，不再散落在各 agent 目录。

| 目录 | 内容 |
|------|------|
| `specs/` | 宪法、计划、Spec-Kit、代码 |
| `reviews/` | 扫描报告、一致性复核、Step3 双审 |
| `arbitration/` | 仲裁结论 |
| `research/` | NotebookLM 历史经验调研 |
| `test/` | 测试报告 |
| `docs/` | 交付文档 |

---

## 流程详解

### Step 1: 任务分级与类型分析

**执行者**: Main（小光）  
**输入**: 用户需求  
**输出**: 
- 级别判定（L1/L2/L3）
- 类型判定（Type A/B）
- 是否需要三模型交叉审核

**分级标准**：
- **L1**（低风险）：单文件修改、普通 bug、小功能补丁
- **L2**（中风险）：跨模块改动、中等复杂度功能
- **L3**（高风险）：安全/权限、数据迁移、支付/交易、复杂架构决策

**类型判定**：
- **Type A**（业务/架构）：业务逻辑、架构设计、API 设计
- **Type B**（算法/性能）：算法优化、性能调优、数据结构

---

### Step 1.5: Constitution-First 打磨层（NotebookLM 深度参与）

#### Step 1.5A: Gemini 扫描

**执行者**: Gemini（织梦）  
**模型**: `gemini/gemini-3.1-pro-preview`  
**Thinking**: medium (L1/L2) / high (L3)  
**输入**: 任务描述  
**输出**: 
- 问题清单（需求歧义点）
- 盲点清单（容易忽略的地方）
- 待验证假设（依赖的前提条件）

**保存位置**: `intel/collaboration/starchain/reviews/scan-YYYYMMDD_HHMMSS.md`

#### Step 1.5B: NotebookLM 深度研究（提前介入，新位置）

**执行者**: NotebookLM（珊瑚）  
**模型**: `anthropic/claude-opus-4-6`（内部调用 NotebookLM）  
**Thinking**: high  
**输入**: Gemini 扫描结果  
**查询**: starchain-knowledge notebook  
**输出**: 实施建议文档
- 历史经验（类似任务的实施经验）
- 常见坑点（容易踩的坑和规避方法，作为宪法禁止项的证据）
- 推荐路径（推荐的技术路径和理由，作为宪法范围的依据）
- 参考实现（可以参考的代码片段或模板）
- 宪法建议（基于历史教训的约束建议）

**保存位置**: `intel/collaboration/starchain/research/implementation-advice-YYYYMMDD_HHMMSS.md`

**核心价值**：
- 为宪法制定提供历史证据
- 避免重复踩坑
- 规则基于数据，而非主观判断

#### Step 1.5C: OpenAI 宪法（基于证据）

**执行者**: OpenAI（GPT-5.4）  
**模型**: `openai/gpt-5.4`  
**Thinking**: medium (L1) / high (L2/L3)  
**输入**: Gemini 扫描结果 + NotebookLM 研究  
**输出**: 开发宪法（1-2 页核心约束，融合历史经验）
- 范围定义（基于 NotebookLM 推荐路径）
- 判定标准
- 禁止项（基于 NotebookLM 常见坑点）
- 证据门槛
- 输出格式

**保存位置**: `intel/collaboration/starchain/specs/constitution-YYYYMMDD_HHMMSS.md`

**改进点**：宪法不再是主观制定，而是基于历史数据和证据

#### Step 1.5D: Claude 计划（基于最佳实践）

**执行者**: Claude（小克）  
**模型**: `anthropic/claude-opus-4-6`  
**Thinking**: medium (L1/L2) / high (L3)  
**输入**: 宪法 + NotebookLM 研究  
**输出**: 实施计划
- 技术路径（参考 NotebookLM 推荐路径）
- 实施步骤
- 风险控制（基于 NotebookLM 常见坑点）
- 验证方式

**保存位置**: `intel/collaboration/starchain/specs/plan-YYYYMMDD_HHMMSS.md`

**改进点**：计划基于历史最佳实践，而非从零开始

#### Step 1.5E: Gemini 一致性复核

**执行者**: Gemini（织梦）  
**模型**: `gemini/gemini-3.1-pro-preview`  
**Thinking**: medium  
**输入**: 宪法 + 计划  
**输出**: 一致性判定
- ALIGN：完全对齐
- DRIFT：轻微偏离
- MAJOR_DRIFT：严重偏离

**保存位置**: `intel/collaboration/starchain/reviews/consistency-YYYYMMDD_HHMMSS.md`

#### Step 1.5F: 仲裁（按需）

**触发条件**：
- Gemini 判定 MAJOR_DRIFT
- L3 级任务强制仲裁
- 宪法与计划有明显冲突

**执行者**: OpenAI 或 Claude（按独立性规则）  
**独立性规则**：
- OpenAI 制定宪法 → Claude 仲裁
- Claude 制定计划 → OpenAI 仲裁

**输出**: GO / REVISE / BLOCK

**保存位置**: `intel/collaboration/starchain/arbitration/decision-YYYYMMDD_HHMMSS.md`

#### Step 1.5G: Brainstorming 落地 Spec-Kit（动态模型）

**执行者**: Brainstorming  
**模型**: 动态选择
- **L1/L2**: `anthropic/claude-sonnet-4-6`（速度优先）
- **L3**: `anthropic/claude-opus-4-6`（质量优先）

**Thinking**: medium (L1/L2) / high (L3)  
**输入**: 宪法 + 计划 + NotebookLM 研究  
**输出**: Spec-Kit 四件套
- `spec.md`：技术规格
- `plan.md`：实施计划
- `tasks.md`：开发任务清单
- `research.md`：技术调研

**保存位置**: `intel/collaboration/starchain/specs/spec-kit-YYYYMMDD_HHMMSS/`

**改进点**：
- 动态模型选择，降本 30-40%
- Spec-Kit 有完整的历史经验支撑
- 判定标准
- 禁止项
- 证据门槛
- 输出格式

**保存位置**: `intel/collaboration/starchain/specs/constitution-YYYYMMDD_HHMMSS.md`

#### Step 1.5C: Claude 计划

**执行者**: Claude（小克）  
**模型**: `anthropic/claude-opus-4-6`  
**Thinking**: medium (L1/L2) / high (L3)  
**输入**: 宪法  
**输出**: 实施计划
- 技术路径
- 实施步骤
- 风险控制
- 验证方式

**保存位置**: `intel/collaboration/starchain/specs/plan-YYYYMMDD_HHMMSS.md`

#### Step 1.5D: Gemini 一致性复核

**执行者**: Gemini（织梦）  
**模型**: `gemini/gemini-3.1-pro-preview`  
**Thinking**: medium  
**输入**: 宪法 + 计划  
**输出**: 一致性判定
- ALIGN：完全对齐
- DRIFT：轻微偏离
- MAJOR_DRIFT：严重偏离

**保存位置**: `intel/collaboration/starchain/reviews/consistency-YYYYMMDD_HHMMSS.md`

#### Step 1.5E: 仲裁（按需）

**触发条件**：
- Gemini 判定 MAJOR_DRIFT
- L3 级任务强制仲裁
- 宪法与计划有明显冲突

**执行者**: OpenAI 或 Claude（按独立性规则）  
**独立性规则**：
- OpenAI 制定宪法 → Claude 仲裁
- Claude 制定计划 → OpenAI 仲裁

**输出**: GO / REVISE / BLOCK

**保存位置**: `intel/collaboration/starchain/arbitration/decision-YYYYMMDD_HHMMSS.md`

---


**执行者**: Coding  
**模型**: Type A: `anthropic/claude-sonnet-4-6` / Type B: `openai/gpt-5.4`  
**Thinking**: medium  
**输入**: Spec-Kit  
**输出**: 代码 + Step 2.5 冒烟测试结果

**保存位置**: `intel/collaboration/starchain/specs/coding-YYYYMMDD_HHMMSS/`

---

### Step 3: 双审（Claude + Gemini）

**执行方式**: Main 并行 spawn Claude 和 Gemini

#### Claude 主审查

**执行者**: Claude（小克）  
**模型**: `anthropic/claude-opus-4-6`  
**Thinking**: medium (L1/L2) / high (L3)  
**输入**: 代码 + Spec-Kit  
**输出**: 审查报告（是否符合规格和宪法）

**保存位置**: `intel/collaboration/starchain/reviews/review-step3-YYYYMMDD_HHMMSS.md`

#### Gemini Adversarial Review

**执行者**: Gemini（织梦）  
**模型**: `gemini/gemini-3.1-pro-preview`  
**Thinking**: medium  
**输入**: 代码 + Spec-Kit  
**输出**: 找漏洞、找风险、找不符合宪法的地方

**保存位置**: `intel/collaboration/starchain/reviews/review-step3-gemini-YYYYMMDD_HHMMSS.md`

#### 仲裁（按需）

**触发条件**：
- Claude 和 Gemini 判定相反
- 有明显分歧点（≥2 个）
- 关键问题分歧

**执行者**: OpenAI（按独立性规则）  
**输出**: GO / REVISE / BLOCK

---

### Step 4: 修复循环（max 3 rounds）

**执行者**: Main 编排  

**循环内容**：
1. **Brainstorming 出修复方案**
   - 模型：动态选择
     - Round 1-2: `anthropic/claude-sonnet-4-6`（快速迭代）
     - Round 3: `anthropic/claude-opus-4-6`（最后一搏，需要创新）
   - Thinking: medium (Round 1-2) / high (Round 3)
   - 输入：审查报告 + 代码
   - 输出：修复方案

2. **Coding 修复**
   - 模型：Type A: sonnet / Type B: gpt-5.4
   - 输入：修复方案
   - 输出：修复后的代码

3. **Main 并行 spawn Gemini + Claude 双审**
   - 重新审查修复后的代码

**退出条件**：
- 双审通过
- 达到 3 轮上限

**改进点**：
- Round 1-2 用 Sonnet 快速迭代
- Round 3 用 Opus 创新突破
- 降本同时保证质量

---

### Step 5: Test 测试

**执行者**: Test  
**模型**: Type A: `openai/gpt-5.4` / Type B: `anthropic/claude-sonnet-4-6`  
**Thinking**: medium  
**输入**: 代码 + Spec-Kit  
**输出**: 测试报告

**保存位置**: `intel/collaboration/starchain/test/test-YYYYMMDD_HHMMSS/`

---

### Step 5.5: Epoch 诊断与仲裁（按需）

**触发条件**：
- 测试失败
- 达到 Epoch 上限（max 3）

**执行流程**：

1. **Gemini 诊断**
   - 模型：`gemini/gemini-3.1-pro-preview`
   - Thinking: high
   - 输入：测试报告 + 代码
   - 输出：诊断报告（为什么失败）

2. **Claude 独立复核**
   - 模型：`anthropic/claude-opus-4-6`
   - Thinking: high
   - 输入：Gemini 诊断 + 测试报告
   - 输出：复核报告（是否同意诊断）

3. **Brainstorming 回滚决策**
   - 模型：`anthropic/claude-opus-4-6`（始终用 Opus，关键决策）
   - Thinking: high
   - 输入：诊断 + 复核 + 历史记录
   - 输出：回滚决策（继续 vs 回滚到哪一步）

4. **OpenAI/Claude 仲裁（按需）**
   - 触发条件：Gemini 和 Claude 诊断严重分歧
   - 模型：`openai/gpt-5.4` 或 `anthropic/claude-opus-4-6`（按独立性规则）
   - Thinking: high
   - 输出：最终决策

**改进点**：
- Brainstorming 回滚决策始终用 Opus
- 关键决策不能因为降本而降低质量

---

### Step 6: 文档生成

**执行者**: Docs  
**模型**: `minimax/MiniMax-M2.5`  
**Thinking**: medium  
**输入**: 代码 + Spec-Kit  
**输出**: 交付文档

**保存位置**: `intel/collaboration/starchain/docs/deliverable-YYYYMMDD_HHMMSS/`

---

### Step 7: 汇总交付

**执行者**: Main（小光）  
**输入**: 所有 agent 产物  
**输出**: 
- 汇总报告
- 通知晨星（Telegram DM: 1099011886）
- 通知监控群（-5131273722）

---

## 独立性规则（与星鉴统一）

**原则**：谁参与主要决策，谁就不做最终仲裁

### 星链规则

- **OpenAI 制定宪法** → Claude 仲裁计划分歧
- **Claude 主方案** → OpenAI 仲裁实施分歧
- **Gemini 始终不做最终仲裁**，只做证据和反方意见

### 星鉴规则

- **OpenAI 制定宪法** → Claude 仲裁方案分歧
- **Claude 复核方案** → OpenAI 仲裁复核分歧
- **Gemini 始终不做最终仲裁**

---

## 通知规范

### 核心原则

sub-agent 执行 agent 自己推群（职能群 + 监控群），main 统一监控群兜底。

### 根因

`sessions_spawn` 的 isolated session 有 Telegram 群 binding，sub-agent 可直接调用 message 工具推群。工具配置已设为 `alsoAllow: [message]`。

### 通知类型（三类必须覆盖）

| 类型 | 触发时机 | 发往 |
|------|---------|------|
| **START** | agent 开始执行本步骤时 | 职能群 + 监控群 |
| **COMPLETION** | agent 完成本步骤时（含结果摘要） | 职能群 + 监控群 |
| **FAILURE** | agent 遇到错误/卡点时 | 职能群 + 监控群 |

### 通知内容要求
- START/COMPLETION 必须包含：步骤名称、本步骤做了什么、下一步是什么
- FAILURE 必须包含：步骤名称、错误原因、已尝试的解决措施
- 不得只发"done"、"开始"等空内容

### main 兜底规则
- 负责补发缺失通知
- 负责最终交付通知
- 负责告警通知

---

## 完成判定

完成信号必须同时满足：
- ✅ 正式文件已写入对应 agent 自己的目录
- ✅ 返回结构化回执（不能只写 `done`）
- ✅ 关键阶段通知已发出或由 main 兜底补发

**⚠️ 缺一项都不算真正完成**

---

## Model Baseline

| Step | 默认模型/执行者 | Thinking Level | 说明 |
|------|------------------|----------------|------|
| Step 1 | `main / opus` | high | 判断任务类型、复杂度、是否需要仲裁 |
| Step 1.5A | `gemini` | medium (L1/L2) / high (L3) | 扫描问题 |
| Step 1.5B | `notebooklm / opus` | high | 深度研究（提前介入） |
| Step 1.5C | `openai (gpt)` | medium (L1) / high (L2/L3) | 宪法制定（基于证据） |
| Step 1.5D | `claude / opus` | medium (L1/L2) / high (L3) | 实施计划（基于最佳实践） |
| Step 1.5E | `gemini` | medium | 一致性复核 |
| Step 1.5F | `openai/claude`（按需） | high | 仲裁 |
| Step 1.5G | `brainstorming / sonnet(L1/L2) or opus(L3)` | medium (L1/L2) / high (L3) | Spec-Kit 落地（动态模型） |
| Step 2 | `coding / sonnet(A) or gpt(B)` | medium | 开发 |
| Step 3 | `claude / opus` + `gemini` | medium (L1/L2) / high (L3) | 双审 |
| Step 4 | `brainstorming / sonnet(R1-2) or opus(R3)` + `coding` | medium (R1-2) / high (R3) | 修复循环（动态模型） |
| Step 5 | `test / gpt(A) or sonnet(B)` | medium | 测试 |
| Step 5.5 | `gemini + claude + brainstorming(opus) + openai/claude` | high | Epoch 诊断与仲裁（Brainstorming 始终 Opus） |
| Step 6 | `docs / minimax` | medium | 文档生成 |
| Step 7 | `main / opus` | high | 汇总与最终通知 |

**动态模型策略**：
- **Step 1.5G (Spec-Kit)**: L1/L2 用 sonnet，L3 用 opus
- **Step 4 (修复循环)**: Round 1-2 用 sonnet，Round 3 用 opus
- **Step 5.5 (回滚决策)**: Brainstorming 始终用 opus（关键决策）

**预期降本**: 35-45%（相比全部使用 opus）

---

## Spawn 规范

所有 agent 一律用 `openclaw agent --agent <agent-id> --task "<任务>"` 调用。

### Spawn 重试机制

任何 agent spawn 失败时（包括 LLM request timed out、503、网络错误等），launcher 必须自动重试：

1. **第一次失败** → 立即重试（相同参数）
2. **第二次失败** → 等待 10 秒后重试
3. **第三次仍失败** → 推送告警到监控群 + 通知晨星 → 标记 BLOCKED

---

## 工具容错与降级

外部工具失败时（nlm-gateway.sh 等）：
1. 发送 Warning 到监控群
2. 跳过失败的工具调用
3. 继续推进到下一步
4. 绝不因工具失败中断流水线

---

## 版本管理

- **小更新不升版本号**：通知细化、措辞修正、轻量步骤调整、同框架内补丁
- **大变动才升新版本**：流程主干、阶段职责、核心门控、模型分工、回滚逻辑、交付结构发生实质变化
- **只保留最近两个版本**：当前版本 + 上一个版本
- **默认入口始终指向最新版本**

---

## 回滚策略

如果 v2.7 出现问题，可以回滚到 v2.6：
1. 恢复 `references/pipeline-v2-6-contract.md` 为默认合约
2. 恢复 Main 直接编排模式
3. 禁用 `starchain-launcher.sh`

---

**合约生效日期**: 2026-03-12  
**合约版本**: v2.7  
**上一版本**: v2.6（回滚参考）
