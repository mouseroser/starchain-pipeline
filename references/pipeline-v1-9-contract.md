# Pipeline v1.9 Contract (Constitution-First StarChain)

Source of truth: `PIPELINE_FLOWCHART_V1_9_EMOJI.md`.


## 1.5) 协作材料规则

- 只要是多 agent 联合工作（流水线尤为明显），除了正式产物之外全部放到 `intel/collaboration/`
- 正式产物（报告、审查结论、修复代码、文档交付等）放各自 agent 目录
- 非正式产物（外部 GitHub 项目、本地镜像、共享分析素材、临时脚本、调研素材等）放 `intel/collaboration/`
- `intel/` 继续作为协作层，遵守单写者原则：一个 agent 负责写 / 更新，其他 agent 只读

## 1) Global State

```text
ReEntry = 0
ReEntry_MAX = 2
Snapshots = {S1, S2, S3}
HALT_TIMEOUT = 30min
G2 = 20 weighted lines
G3 = 30 weighted lines
Weight: critical paths ×2, others ×1
```

## 2) Lane Routing

- Step 1 classifies every task into `L1 / L2 / L3` + `Type A / B / C`.
- Type A (业务/架构): `coding = sonnet/medium`, `review = gpt/high`, `test = gpt/medium`.
- Type B (算法/性能): `coding = gpt/medium`, `review = sonnet/medium`, `test = sonnet/medium`.
- Type C (混合): split into sub-tasks and route through A/B separately.
- L1 → fast lane.
- L2/L3 → mandatory Step 1.5 constitution-first preflight.

## 3) Step 1.5 Constitution-First Preflight (L2/L3 Only)

Step 1.5 in v1.9 is no longer just a spec-kit drafting gate. It is an upgraded preflight chain whose goal is to align problem granularity before implementation planning.

### 3.1 Core Principle

The preflight chain must follow this order:
- `Gemini` first aligns problem granularity and produces a **problem constitution**.
- `Claude Code` produces the execution plan based on that constitution.
- `Gemini` reviews whether the plan drifted from the original problem.
- `GPT` only enters for `L3`, high-risk, or major-drift cases as a risk challenger / arbitration layer.

### 3.2 Lane Rules

- `L1`: skip heavy preflight and use fast lane.
- `L2`: `Gemini → Claude Code → Gemini`
- `L3`: `Gemini → Claude Code → Gemini → GPT`

### 3.3 L2 Standard Chain

1. `Gemini` produces the **problem constitution**.
2. `Claude Code` produces the execution plan from that constitution.
3. `Gemini` reviews the plan and returns one of:
   - `ALIGN`
   - `DRIFT`
   - `MAJOR_DRIFT`

Routing:
- `ALIGN` → Step 2
- `DRIFT` → Claude Code revises once, then Gemini re-reviews
- `MAJOR_DRIFT` → escalate to GPT arbitration layer

### 3.4 L3 Escalated Chain

1. `Gemini` → constitution
2. `Claude Code` → implementation plan
3. `Gemini` → consistency review
4. `GPT` → outputs one of:
   - `GO`
   - `REVISE`
   - `BLOCK`

Routing:
- `GO` → Step 2
- `REVISE` → Claude Code revises once → Gemini re-reviews → re-evaluate
- `BLOCK` → main pushes blocking reason → halt or degraded delivery

### 3.5 Hard Rules for Step 1.5

- No new human confirmation gate is added in the middle.
- Step 1.5 remains fully automatic.
- This chain upgrades Step 1.5 only; it does not replace downstream development / review / test / docs phases.
- `Claude Code` is the main planner.
- `Gemini` is the front alignment + back consistency checker.
- `GPT` is the challenger / arbitration layer, not a mandatory middle hop for every task.

## 4) L1 Fast Lane

- L1 skips the heavy preflight chain.
- Flow:
  1. `coding`
  2. smoke test
  3. single cross-review
  4. final delivery
- If L1 fails to pass after one quick repair path, it must auto-upgrade to L2 and restart from Step 1.5.

## 5) Step 2 Development

- `coding` executes strictly according to the final Step 1.5 plan.
- Step 2 completion creates snapshot `S1`.
- Coding status must be visible in coding functional notifications; main also pushes monitor updates.

## 6) Step 2.5 Smoke Test Gate

Executed by `coding`.

Required checks:
- compile / syntax
- unit-test sanity
- basic functionality

Routing:
- `PASS` → Step 3
- `FAIL` → self-fix max 2 times
- still `FAIL` → Step 4 with logs

## 7) Step 3 Structured Cross-Review

`review` executes structured cross-review with the v1.9 three-model principle:
- Claude / GPT are the main review model pair
- Gemini may be added as diagnosis context when needed
- high-dispute cases escalate to arbitration

### 7.1 Verdict Set

```json
{
  "verdict": "PASS | PASS_WITH_NOTES | NEEDS_FIX",
  "issues": [{ "severity": "...", "category": "...", "file": "...", "line": 0, "description": "...", "suggestion": "..." }]
}
```

### 7.2 Verdict Rules

- `PASS`: no critical, no major, no minor
- `PASS_WITH_NOTES`: no critical, no major, minor ≤ 3
- `NEEDS_FIX`: any critical or any major

### 7.3 Snapshot + Diff Guard

- `PASS` → create `S2` → Step 5
- `PASS_WITH_NOTES`:
  - diff ≤ `G2` → minor fix without re-review → create `S2` → Step 5
  - diff > `G2` → downgrade to `NEEDS_FIX` → Step 4
- `NEEDS_FIX` → Step 4

## 8) Weighted Diff Gates

Formula:

```text
weighted_diff = Σ(changed_lines × weight)
critical paths = 2
others = 1
```

Thresholds:
- `G2 = 20` weighted lines
- `G3 = 30` weighted lines

Usage:
- G2 guards `PASS_WITH_NOTES`
- G3 guards TF return-to-review

## 9) Step 4 Repair Loop

Max 3 rounds.

Every round must include the full context bundle:
- original requirement
- Step 1.5 final plan / constitution
- current diff
- structured review issues
- prior repair diffs / review feedback
- smoke failure logs (if any)

Round progression:
1. `brainstorming sonnet/medium + coding gpt/medium`
2. `brainstorming sonnet/medium + coding gpt/medium`
3. `brainstorming opus/high + coding gpt/xhigh`

After round 3 still `NEEDS_FIX` → Step 5.5.

## 10) Step 5 Test + TF Path

- Step 5 pass → create `S3` → Step 6
- Step 5 fail → TF path

TF path:
- TF-1: coding direct fix + test rerun
- TF-2: brainstorming sonnet/medium + coding + test rerun
- TF-3: brainstorming opus/high + coding + test rerun + regression

Rules:
- TF-1 / TF-2 diff ≤ `G3` → full regression → Step 6
- TF-1 / TF-2 diff > `G3` → Step 3 (`ReEntry++`)
- TF-3 always returns to Step 3 (`ReEntry++`)
- if `ReEntry > ReEntry_MAX` → force Step 5.5

## 11) Bounded Loop Rules

- `ReEntry_MAX = 2`
- Step 4 max 3 rounds
- Step 5.5 max 3 Epochs

No silent infinite loops are allowed.

## 12) Step 5.5 Epoch Fallback

Entry conditions:
- Step 4 round 3 still fails
- TF-3 still fails

Required sequence:
1. `Gemini` produces diagnosis memo
2. `brainstorming opus/high` produces rollback decision

Rollback options:
- rollback to `S1`
- rollback to `S2`
- continue without rollback

If `Epoch > 3`:
- reach the only halt point
- start `HALT_TIMEOUT = 30min`
- on timeout:
  - `GPT/high` diagnosis
  - degraded delivery from `S2`
  - notify monitor + 晨星

## 13) Step 6 Documentation

Skip for L1.

Required sequence:
1. `notebooklm` queries templates / history knowledge
2. `Gemini` drafts release notes / FAQ outline
3. `docs` generates final docs

Inputs:
- final diff
- original requirement
- review summary
- Step 1.5 artifacts / constitution plan

## 14) Step 7 Final Delivery

Executed by `main`.

Final delivery must include:
- completed scope
- changed files
- test results
- review conclusion
- level (`L1/L2/L3`)
- type (`A/B/C`)
- delivery status (`normal | degraded`)
- constitution / plan summary
- snapshot tags (`S1/S2/S3`)

Destinations:
- monitor group
- 晨星 DM

## 15) Tool Resilience & Graceful Degradation

Any external tool / CLI failure (for example `nlm-gateway.sh`) must follow this rule:
1. send warning to monitor group
2. skip the failed tool call and fall back to model reasoning
3. continue pipeline execution

Tool failure must not halt the pipeline by itself.

## 16) Spawn Retry Mechanism

For any spawn failure:
1. retry immediately once
2. retry again after 10 seconds
3. if still failing, notify monitor + 晨星 and mark step `BLOCKED`

A single failed spawn must never directly halt the whole pipeline.

## 17) Notification Contract

### 17.1 Ownership

- Functional-group visibility belongs primarily to the capable agent itself.
- Monitor-group visibility, missing-message补发, final delivery, and alerts belong to `main`.

### 17.2 Granularity

- simple delegation → start / finish / fail
- multi-agent orchestration → start / key stage / finish / fail
- formal pipeline → stage-by-stage notifications

### 17.3 Step 1.5 Visibility

Step 1.5 should surface:
- Gemini constitution result
- Claude Code plan result
- Gemini consistency result
- GPT arbitration result (when applicable)

A plan-type task must not end with a bare `done`; completion must include actual summary payload.

## 18) Direction-Change Rule

If the same problem remains unsolved after three consecutive attempts along the same direction, the pipeline must switch direction and redefine the problem instead of continuing to double down on the same failed path.

## 19) Three-Model Principle (v1.9)

```text
Gemini = problem granularity alignment + consistency review + diagnosis memo
Claude Code = main planner / execution planner
GPT = challenger / arbitration / pressure test

L2: Gemini → Claude Code → Gemini
L3: Gemini → Claude Code → Gemini → GPT

Development and downstream review/test still follow Type A / B routing.
```
