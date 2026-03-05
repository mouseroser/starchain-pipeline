# Step 4 增量审查模式

## 目的

在 Step 4 修复循环中，R1/R2 使用增量审查（只审查修改部分），R3 使用完整审查，减少审查时间和成本。

---

## 审查模式对比

### 完整审查（Full Review）

**适用场景**：
- Step 3 首次审查
- Step 4 R3（最后一轮）
- TF-3 返回 Step 3

**审查范围**：
- 所有变更代码
- 上下文代码
- 依赖关系
- 架构影响

**优点**：全面，不遗漏副作用
**缺点**：耗时长，成本高

---

### 增量审查（Incremental Review）

**适用场景**：
- Step 4 R1/R2
- TF-1/TF-2 返回 Step 3（可选）

**审查范围**：
- 本轮修改的代码（git diff）
- 直接相关的上下文
- 针对本轮 issues 的修复

**优点**：快速，成本低
**缺点**：可能遗漏副作用

---

## 增量审查实施

### 审查范围定义

```python
def get_incremental_review_scope(round_number, previous_issues, current_diff):
    """定义增量审查范围"""
    
    # 1. 提取本轮修改的文件
    changed_files = extract_changed_files(current_diff)
    
    # 2. 提取本轮修改的函数/类
    changed_symbols = extract_changed_symbols(current_diff)
    
    # 3. 映射到本轮要修复的 issues
    target_issues = map_issues_to_changes(previous_issues, changed_symbols)
    
    return {
        "mode": "incremental",
        "round": round_number,
        "changed_files": changed_files,
        "changed_symbols": changed_symbols,
        "target_issues": target_issues,
        "context_lines": 10  # 上下文行数
    }
```

### 增量 Verdict 格式

```json
{
  "mode": "incremental",
  "round": 1,
  "verdict": "PASS | PASS_WITH_NOTES | NEEDS_FIX",
  "target_issues": [
    {
      "issue_id": "issue-001",
      "description": "逻辑错误：登录验证不完整",
      "status": "FIXED | PARTIALLY_FIXED | NOT_FIXED",
      "verification": "修复正确，逻辑完整"
    },
    {
      "issue_id": "issue-002",
      "description": "性能问题：数据库查询未优化",
      "status": "FIXED",
      "verification": "已添加索引，查询优化"
    }
  ],
  "new_issues": [
    {
      "severity": "minor",
      "category": "style",
      "description": "变量命名不规范",
      "file": "src/auth.py",
      "line": 45
    }
  ],
  "side_effects_check": "PASS | WARNING",
  "side_effects": []
}
```

### 副作用检查

即使是增量审查，也需要快速检查副作用：

```python
def check_side_effects(changed_symbols, codebase):
    """快速检查副作用"""
    
    side_effects = []
    
    for symbol in changed_symbols:
        # 1. 检查调用者（谁调用了这个函数）
        callers = find_callers(symbol, codebase)
        if len(callers) > 5:
            side_effects.append({
                "type": "high_coupling",
                "symbol": symbol,
                "callers_count": len(callers),
                "warning": "修改影响多个调用者，建议完整审查"
            })
        
        # 2. 检查接口变更（函数签名是否改变）
        if signature_changed(symbol):
            side_effects.append({
                "type": "interface_change",
                "symbol": symbol,
                "warning": "接口变更，可能影响调用者"
            })
        
        # 3. 检查数据结构变更
        if data_structure_changed(symbol):
            side_effects.append({
                "type": "data_structure_change",
                "symbol": symbol,
                "warning": "数据结构变更，可能影响序列化/存储"
            })
    
    return side_effects
```

---

## 审查模式切换规则

### Step 4 审查模式

| 轮次 | 审查模式 | 原因 |
|------|---------|------|
| R1 | 增量审查 | 快速验证修复，节省成本 |
| R2 | 增量审查 | 继续快速迭代 |
| R3 | 完整审查 | 最后一轮，全面检查，防止遗漏 |

### 强制完整审查触发条件

即使在 R1/R2，以下情况也触发完整审查：

| 触发条件 | 原因 |
|---------|------|
| 副作用检查 WARNING | 发现高耦合或接口变更 |
| 修改文件数 > 5 | 变更范围过大 |
| 修改行数 > 100 | 变更量过大 |
| 新增 critical issue | 引入严重问题 |

```python
def should_force_full_review(incremental_result):
    """判断是否强制完整审查"""
    
    # 1. 副作用检查 WARNING
    if incremental_result["side_effects_check"] == "WARNING":
        return True, "发现副作用风险"
    
    # 2. 修改范围过大
    if len(incremental_result["changed_files"]) > 5:
        return True, "修改文件数超过 5"
    
    # 3. 修改量过大
    if incremental_result["changed_lines"] > 100:
        return True, "修改行数超过 100"
    
    # 4. 新增严重问题
    critical_issues = [i for i in incremental_result["new_issues"] 
                      if i["severity"] == "critical"]
    if critical_issues:
        return True, "引入 critical 问题"
    
    return False, None
```

---

## Verdict 合并

### 合并逻辑

增量审查的 verdict 需要与之前的 issues 合并：

```python
def merge_verdicts(previous_issues, incremental_verdict):
    """合并增量 verdict 到全局 issues"""
    
    merged_issues = []
    
    # 1. 更新已修复的 issues
    for issue in previous_issues:
        target = find_target_issue(issue, incremental_verdict["target_issues"])
        
        if target:
            if target["status"] == "FIXED":
                # 标记为已修复，不再包含在 merged_issues
                continue
            elif target["status"] == "PARTIALLY_FIXED":
                # 更新描述，继续包含
                issue["description"] = target["verification"]
                merged_issues.append(issue)
            else:  # NOT_FIXED
                # 保持原样
                merged_issues.append(issue)
        else:
            # 本轮未涉及，保持原样
            merged_issues.append(issue)
    
    # 2. 添加新发现的 issues
    merged_issues.extend(incremental_verdict["new_issues"])
    
    return merged_issues
```

### 合并后的 Verdict

```json
{
  "mode": "merged",
  "round": 1,
  "verdict": "NEEDS_FIX",
  "issues": [
    {
      "issue_id": "issue-003",
      "severity": "major",
      "description": "安全问题：密码未加密（未修复）",
      "status": "NOT_FIXED"
    },
    {
      "issue_id": "issue-004",
      "severity": "minor",
      "description": "变量命名不规范（新发现）",
      "status": "NEW"
    }
  ],
  "fixed_issues": ["issue-001", "issue-002"],
  "remaining_issues": 2
}
```

---

## 实施流程

### Step 4 R1/R2 流程

```markdown
1. brainstorming 产出修复方案
2. coding 执行修复
3. **增量审查**：
   - review 接收：
     - 原始需求
     - 前轮 issues JSON
     - 本轮 diff
     - 修复方案
   - review 执行增量审查：
     - 只审查本轮修改
     - 验证目标 issues 是否修复
     - 快速检查副作用
   - review 输出增量 verdict
4. main 合并 verdict：
   - 更新全局 issues JSON
   - 判断是否需要下一轮
5. **强制完整审查检查**：
   - 如果触发条件 → 降级为完整审查
   - 重新执行 Step 3 完整审查流程
```

### Step 4 R3 流程

```markdown
1. brainstorming 产出修复方案（opus/high）
2. coding 执行修复（codex/xhigh）
3. **完整审查**：
   - review 接收完整 context
   - review 执行完整审查（所有代码）
   - review 输出完整 verdict
4. main 判断：
   - PASS/PASS_WITH_NOTES → Step 5
   - NEEDS_FIX → Step 5.5
```

---

## 成本和时间对比

### 审查成本估算

| 审查模式 | Token 消耗 | 时间 | 适用场景 |
|---------|-----------|------|---------|
| 完整审查 | 30k ~ 80k | 3-8 分钟 | Step 3, R3, TF-3 |
| 增量审查 | 10k ~ 30k | 1-3 分钟 | R1, R2 |
| **节省** | **60-70%** | **60-70%** | R1/R2 使用增量 |

### Step 4 总成本对比

**传统模式（3 轮完整审查）**：
```
R1: 30k + R2: 30k + R3: 30k = 90k tokens
```

**增量模式（R1/R2 增量 + R3 完整）**：
```
R1: 15k + R2: 15k + R3: 30k = 60k tokens
节省：30k tokens（33%）
```

---

## 集成到流水线

### pipeline-v1-8-contract.md 更新

```markdown
## 6) Step 4 Repair Loop

- Max 3 rounds. Each round must carry full context bundle.
- Context bundle: original requirement + current diff + review JSON + prior round diffs/feedback + smoke logs.
- **Incremental review mode (R1/R2)**:
  - Review only current round changes (git diff)
  - Verify target issues fixed
  - Quick side-effects check
  - Force full review if: side-effects WARNING, files>5, lines>100, new critical issue
- **Full review mode (R3)**:
  - Review all changes
  - Comprehensive check
- Round progression:
  1. brainstorming Sonnet/medium + coding Codex/medium → **incremental review**
  2. brainstorming Sonnet/medium + coding Codex/medium → **incremental review**
  3. brainstorming Opus/high + coding Codex/xhigh → **full review**
- After round 3 still `NEEDS_FIX` → Step 5.5.
```

### review AGENTS.md 更新

```markdown
### Step 4: 修复循环审查（增量模式）

**R1/R2 增量审查**：
1. 接收 context:
   - 前轮 issues JSON
   - 本轮 diff
   - 修复方案
2. 执行增量审查（参考 `references/incremental-review.md`）:
   - 只审查本轮修改
   - 验证目标 issues 是否修复
   - 快速检查副作用
3. 输出增量 verdict:
   - target_issues 状态（FIXED/PARTIALLY_FIXED/NOT_FIXED）
   - new_issues（本轮新发现）
   - side_effects_check（PASS/WARNING）
4. **强制完整审查检查**:
   - 副作用 WARNING → 降级为完整审查
   - 修改范围过大 → 降级为完整审查

**R3 完整审查**：
- 执行完整审查（所有代码）
- 输出完整 verdict
```

---

## 监控和统计

### 增量审查效果统计

```json
{
  "month": "2026-03",
  "total_step4_rounds": 75,
  "by_mode": {
    "incremental": {
      "count": 50,
      "avg_tokens": 18000,
      "avg_time_min": 2.5,
      "forced_full_count": 5
    },
    "full": {
      "count": 25,
      "avg_tokens": 45000,
      "avg_time_min": 6.0
    }
  },
  "savings": {
    "tokens_saved": 1350000,
    "time_saved_min": 175,
    "cost_reduction": 0.40
  },
  "quality_impact": {
    "missed_issues": 2,
    "false_positives": 1,
    "accuracy": 0.96
  }
}
```

### 持续优化

1. **每月回顾**：
   - 分析遗漏的 issues
   - 调整强制完整审查触发条件
   - 优化副作用检查逻辑

2. **季度校准**：
   - 评估增量审查准确率
   - 调整审查范围定义
   - 更新合并逻辑

---

## 注意事项

1. **R3 必须完整审查**：最后一轮不能妥协
2. **副作用检查不能省**：即使增量审查也要快速检查
3. **强制完整审查触发**：发现风险立即降级
4. **Verdict 合并准确**：确保 issues 状态正确更新
5. **质量优先**：节省成本不能以牺牲质量为代价
6. **持续监控**：跟踪遗漏的 issues，优化审查逻辑
