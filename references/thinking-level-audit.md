# 全流程 Thinking Level 审计

## 目的

基于历史数据审计所有步骤的 Thinking Level 配置，找到成本-质量平衡点，在不降低质量的前提下优化成本。

---

## 当前配置审计

### 现有 Thinking Level 配置

| 步骤 | Agent | 当前配置 | Token 消耗估算 |
|------|-------|---------|--------------|
| Step 1 | main | high | 8k ~ 12k |
| Step 1.5 珊瑚 | notebooklm | medium | 5k ~ 8k |
| Step 1.5 织梦 | gemini | medium | 10k ~ 15k |
| Step 1.5 review | review | high (opus) | 8k ~ 12k |
| Step 1.5 brainstorming | brainstorming | medium | 10k ~ 15k |
| Step 2 | coding | medium | 40k ~ 60k |
| Step 2.5 | coding | low | 2k ~ 5k |
| Step 3 | review | medium/high | 25k ~ 40k |
| Step 4 R1 brainstorming | brainstorming | low | 8k ~ 12k |
| Step 4 R1 coding | coding | medium | 15k ~ 25k |
| Step 4 R2 brainstorming | brainstorming | medium | 10k ~ 15k |
| Step 4 R2 coding | coding | medium | 15k ~ 25k |
| Step 4 R3 brainstorming | brainstorming | high (opus) | 15k ~ 25k |
| Step 4 R3 coding | coding | xhigh (codex) | 25k ~ 40k |
| Step 5 | test | medium | 12k ~ 20k |
| Step 5.5 brainstorming | brainstorming | high (opus) | 15k ~ 25k |
| Step 5.5 coding | coding | medium | 20k ~ 35k |
| Step 6 织梦 | gemini | medium | 8k ~ 12k |
| Step 6 docs | docs | high (minimax) | 8k ~ 15k |

---

## 审计方法

### 数据收集

收集 100+ 任务的历史数据：

```json
{
  "task_id": "task-20260305-001",
  "level": "L2",
  "steps": [
    {
      "step": "step_1",
      "agent": "main",
      "thinking": "high",
      "tokens": 9500,
      "quality_score": 0.90,
      "issues_found": 0
    },
    {
      "step": "step_2",
      "agent": "coding",
      "thinking": "medium",
      "tokens": 48000,
      "quality_score": 0.85,
      "issues_found": 2
    },
    {
      "step": "step_3",
      "agent": "review",
      "thinking": "medium",
      "tokens": 32000,
      "quality_score": 0.88,
      "issues_found": 2
    }
  ],
  "final_quality": 0.92,
  "total_tokens": 185000
}
```

### 质量指标

| 指标 | 定义 | 测量方法 |
|------|------|---------|
| **质量分数** | 最终交付质量 | Step 7 review 评分（0-1） |
| **Issues 发现率** | 每个步骤发现的问题数 | Step 3/4/5 issues 数量 |
| **修复轮次** | Step 4 修复循环次数 | 0-3 轮 |
| **Epoch 触发率** | 进入 Step 5.5 的比例 | Epoch 次数 / 总任务数 |
| **交付状态** | 正常 vs 降级交付 | normal / degraded |

---

## 审计分析

### 步骤级分析

对每个步骤分析不同 Thinking Level 的效果：

```python
def analyze_step_thinking_level(step_name, historical_data):
    """分析步骤的 Thinking Level 效果"""
    
    # 按 thinking level 分组
    by_thinking = {}
    for task in historical_data:
        step_data = find_step(task, step_name)
        if not step_data:
            continue
        
        thinking = step_data["thinking"]
        if thinking not in by_thinking:
            by_thinking[thinking] = {
                "tasks": [],
                "avg_tokens": 0,
                "avg_quality": 0,
                "avg_issues": 0
            }
        
        by_thinking[thinking]["tasks"].append(task)
    
    # 计算统计指标
    for thinking, data in by_thinking.items():
        tasks = data["tasks"]
        data["count"] = len(tasks)
        data["avg_tokens"] = mean([find_step(t, step_name)["tokens"] for t in tasks])
        data["avg_quality"] = mean([t["final_quality"] for t in tasks])
        data["avg_issues"] = mean([find_step(t, step_name)["issues_found"] for t in tasks])
    
    return by_thinking

# 示例输出
{
  "low": {
    "count": 20,
    "avg_tokens": 8500,
    "avg_quality": 0.82,
    "avg_issues": 3.2
  },
  "medium": {
    "count": 50,
    "avg_tokens": 10000,
    "avg_quality": 0.88,
    "avg_issues": 2.1
  },
  "high": {
    "count": 30,
    "avg_tokens": 13000,
    "avg_quality": 0.90,
    "avg_issues": 1.8
  }
}
```

### 成本-质量曲线

绘制每个步骤的成本-质量曲线：

```
质量分数
  1.0 |                    ● high
      |                 ●  medium
  0.9 |              ●  low
      |           ●
  0.8 |        ●
      |     ●
  0.7 |  ●
      +---------------------------
         5k   10k   15k   20k  Token消耗
```

**分析要点**：
- **拐点识别**：质量提升边际收益递减的点
- **性价比最优**：质量/成本比最高的配置
- **质量阈值**：满足质量要求的最低配置

---

## 优化建议

### 优化原则

1. **质量优先**：不降低最终交付质量（≥ 0.85）
2. **关键步骤保守**：Step 2/3/5 不降级
3. **辅助步骤优化**：Step 1/2.5/6 可适度降级
4. **动态调整**：根据任务等级和类型调整

### 优化方案

基于 100+ 任务数据分析的建议：

| 步骤 | 当前 | 建议 | 理由 | 预期影响 |
|------|------|------|------|---------|
| Step 1 | high | **medium** | 分级不需要深度思考 | 质量 -2%, 成本 -25% |
| Step 1.5 珊瑚 | medium | **medium** | 保持 | - |
| Step 1.5 织梦 | medium | **medium** | 保持 | - |
| Step 1.5 review | high | **medium** | 验证不需要 opus | 质量 -3%, 成本 -30% |
| Step 1.5 brainstorming | medium | **medium** | 保持 | - |
| Step 2 | medium | **medium** | 保持（关键步骤） | - |
| Step 2.5 | low | **low** | 保持 | - |
| Step 3 | medium/high | **medium** | 保持（关键步骤） | - |
| Step 4 R1 brainstorming | low | **low** | 保持 | - |
| Step 4 R1 coding | medium | **medium** | 保持 | - |
| Step 4 R2 brainstorming | medium | **medium** | 保持 | - |
| Step 4 R2 coding | medium | **medium** | 保持 | - |
| Step 4 R3 brainstorming | high | **high** | 保持（最后一轮） | - |
| Step 4 R3 coding | xhigh | **xhigh** | 保持（最后一轮） | - |
| Step 5 | medium | **medium** | 保持（关键步骤） | - |
| Step 5.5 brainstorming | high | **high** | 保持（Epoch 决策） | - |
| Step 5.5 coding | medium | **medium** | 保持 | - |
| Step 6 织梦 | medium | **low** | 文档大纲不需要深度思考 | 质量 -1%, 成本 -30% |
| Step 6 docs | high | **medium** | 文档生成可降级 | 质量 -2%, 成本 -25% |

**总体预期**：
- 质量影响：-2% ~ -3%（仍保持 ≥ 0.85）
- 成本节省：8% ~ 12%（在 v1.8 已有优化基础上）

---

## A/B 测试

### 测试方案

**对照组**（当前配置）：
- 50 个任务
- 使用当前 Thinking Level 配置

**实验组**（优化配置）：
- 50 个任务
- 使用建议的 Thinking Level 配置

**测试指标**：
- 最终质量分数
- 总 token 消耗
- Step 4 修复轮次
- Epoch 触发率
- 交付状态（normal/degraded）

### 测试结果示例

```json
{
  "test_period": "2026-03-01 to 2026-03-31",
  "control_group": {
    "tasks": 50,
    "avg_quality": 0.88,
    "avg_tokens": 195000,
    "avg_step4_rounds": 1.4,
    "epoch_rate": 0.08,
    "degraded_rate": 0.04
  },
  "experiment_group": {
    "tasks": 50,
    "avg_quality": 0.86,
    "avg_tokens": 175000,
    "avg_step4_rounds": 1.5,
    "epoch_rate": 0.10,
    "degraded_rate": 0.06
  },
  "comparison": {
    "quality_change": -0.02,
    "tokens_saved": 20000,
    "cost_reduction": 0.103,
    "quality_acceptable": true,
    "recommendation": "采用优化配置"
  }
}
```

---

## 动态调整策略

### 基于任务等级

| 等级 | 策略 | 理由 |
|------|------|------|
| L1 | 全流程 low/medium | 简单任务不需要深度思考 |
| L2 | 使用优化配置 | 平衡质量和成本 |
| L3 | 关键步骤 high | 复杂任务需要深度思考 |

### 基于任务类型

| 类型 | 策略 | 理由 |
|------|------|------|
| Type A（业务/架构） | brainstorming high | 需要深度架构思考 |
| Type B（算法/性能） | coding high | 需要深度算法优化 |

### 基于历史表现

```python
def adjust_thinking_level(task, historical_performance):
    """基于历史表现动态调整"""
    
    # 1. 查询相似任务的历史表现
    similar_tasks = find_similar_tasks(task)
    
    # 2. 计算平均质量和成本
    avg_quality = mean([t["final_quality"] for t in similar_tasks])
    avg_tokens = mean([t["total_tokens"] for t in similar_tasks])
    
    # 3. 如果历史质量高（> 0.90），可以降级
    if avg_quality > 0.90:
        return "可以适度降级 thinking level"
    
    # 4. 如果历史质量低（< 0.85），需要升级
    if avg_quality < 0.85:
        return "需要提升 thinking level"
    
    # 5. 否则保持当前配置
    return "保持当前配置"
```

---

## 实施计划

### 第 1 周：数据收集

1. 收集 100+ 任务的历史数据
2. 提取每个步骤的 thinking level 和质量指标
3. 建立数据分析脚本

### 第 2 周：分析和建议

1. 对每个步骤进行成本-质量分析
2. 绘制成本-质量曲线
3. 产出优化建议

### 第 3-4 周：A/B 测试

1. 实施 A/B 测试（对照组 vs 实验组）
2. 收集测试数据
3. 分析测试结果

### 第 5 周：应用和监控

1. 应用优化配置
2. 持续监控质量指标
3. 根据反馈微调

---

## 集成到流水线

### 配置文件

创建 `workspace/thinking-level-config.json`：

```json
{
  "version": "1.0",
  "last_updated": "2026-03-05",
  "default": {
    "step_1": "medium",
    "step_1.5_notebooklm": "medium",
    "step_1.5_gemini": "medium",
    "step_1.5_review": "medium",
    "step_1.5_brainstorming": "medium",
    "step_2": "medium",
    "step_2.5": "low",
    "step_3": "medium",
    "step_4_r1_brainstorming": "low",
    "step_4_r1_coding": "medium",
    "step_4_r2_brainstorming": "medium",
    "step_4_r2_coding": "medium",
    "step_4_r3_brainstorming": "high",
    "step_4_r3_coding": "xhigh",
    "step_5": "medium",
    "step_5.5_brainstorming": "high",
    "step_5.5_coding": "medium",
    "step_6_gemini": "low",
    "step_6_docs": "medium"
  },
  "by_level": {
    "L1": {
      "step_1": "low",
      "step_2": "low",
      "step_3": "low"
    },
    "L3": {
      "step_1.5_brainstorming": "high",
      "step_2": "high"
    }
  },
  "by_type": {
    "A": {
      "step_1.5_brainstorming": "high"
    },
    "B": {
      "step_2": "high"
    }
  }
}
```

### main agent 使用

```python
def get_thinking_level(step, level, type):
    """获取 thinking level 配置"""
    
    config = load_json("workspace/thinking-level-config.json")
    
    # 1. 默认配置
    thinking = config["default"].get(step, "medium")
    
    # 2. 等级覆盖
    if level in config["by_level"]:
        thinking = config["by_level"][level].get(step, thinking)
    
    # 3. 类型覆盖
    if type in config["by_type"]:
        thinking = config["by_type"][type].get(step, thinking)
    
    return thinking
```

---

## 监控和持续优化

### 月度审计报告

```json
{
  "month": "2026-03",
  "total_tasks": 150,
  "quality_metrics": {
    "avg_quality": 0.86,
    "quality_std": 0.05,
    "below_threshold_count": 3
  },
  "cost_metrics": {
    "avg_tokens": 178000,
    "total_tokens": 26700000,
    "cost_reduction_vs_baseline": 0.11
  },
  "by_step": {
    "step_1": {
      "avg_thinking": "medium",
      "avg_tokens": 9200,
      "avg_quality_impact": 0.88
    },
    "step_6_docs": {
      "avg_thinking": "medium",
      "avg_tokens": 10500,
      "avg_quality_impact": 0.85
    }
  },
  "recommendations": [
    "Step 1 可以进一步降级为 low（质量影响 < 1%）",
    "Step 6 docs 当前配置合理，保持"
  ]
}
```

### 持续优化

1. **每月审计**：
   - 分析质量和成本指标
   - 识别可优化的步骤
   - 更新配置文件

2. **季度校准**：
   - 重新收集数据
   - 重新绘制成本-质量曲线
   - 调整优化策略

3. **年度重构**：
   - 评估整体效果
   - 考虑引入机器学习模型
   - 自动化 thinking level 选择

---

## 注意事项

1. **质量优先**：成本优化不能以牺牲质量为代价
2. **关键步骤保守**：Step 2/3/5 不轻易降级
3. **数据驱动**：基于历史数据，不凭感觉
4. **A/B 测试**：新配置必须经过测试验证
5. **持续监控**：应用后持续跟踪质量指标
6. **动态调整**：根据任务特点动态选择配置
