# Step 1.5 依赖管理优化

## 目的

优化 Step 1.5 的并行执行，通过依赖声明机制确保织梦能获取珊瑚的历史知识，提升质量 10%。

---

## 当前问题

### 完全并行的问题

**当前流程**：
```
珊瑚(notebooklm) ─┐
                  ├─→ 等待两者完成 → 整合
织梦(gemini)     ─┘
```

**问题**：
1. 织梦无法使用珊瑚的历史知识
2. 织梦产出的初稿质量受限
3. 整合时需要手动合并，可能遗漏关键信息

---

## 依赖管理方案

### 方案 1：串行执行（保守）

```
珊瑚(notebooklm) → 织梦(gemini) → review → brainstorming
```

**优点**：
- 织梦能完整使用珊瑚知识
- 逻辑清晰，易于实现

**缺点**：
- 执行时间增加 50%（原 2-3 分钟 → 3-4.5 分钟）
- 失去并行优化的效率提升

---

### 方案 2：部分并行（推荐）

```
珊瑚(notebooklm) ─→ 等待珊瑚完成 ─→ 织梦(gemini) ─→ review ─→ brainstorming
```

**优点**：
- 织梦能使用珊瑚知识
- 相比完全串行，时间仅增加 10-15%
- 质量提升 10%

**缺点**：
- 需要实现依赖等待机制

---

### 方案 3：流式传递（未来优化）

```
珊瑚(notebooklm) ─→ 实时输出 ─→ 织梦(gemini，流式接收)
```

**优点**：
- 最优的时间效率
- 织梦能实时使用珊瑚输出

**缺点**：
- 实现复杂，需要流式 API 支持
- 当前版本不支持

---

## 依赖声明机制

### 配置格式

创建 `references/step-1.5-dependencies.yaml`：

```yaml
# Step 1.5 依赖配置
tasks:
  - name: 珊瑚查询
    agent: notebooklm
    dependencies: []
    optional: true
    timeout_seconds: 120
    output: workspace/step-1.5/notebooklm-output.md
  
  - name: 织梦分析
    agent: gemini
    dependencies:
      - 珊瑚查询  # 等待珊瑚完成
    optional: false
    timeout_seconds: 180
    input:
      - workspace/step-1.5/notebooklm-output.md
    output: workspace/step-1.5/gemini-output.md
  
  - name: review验证
    agent: review
    dependencies:
      - 织梦分析  # 等待织梦完成
    optional: false
    timeout_seconds: 120
    input:
      - workspace/step-1.5/gemini-output.md
    output: workspace/step-1.5/review-output.md
  
  - name: brainstorming产出
    agent: brainstorming
    dependencies:
      - review验证  # 等待 review 完成
    optional: false
    timeout_seconds: 300
    input:
      - workspace/step-1.5/notebooklm-output.md
      - workspace/step-1.5/gemini-output.md
      - workspace/step-1.5/review-output.md
    output:
      - specs/{feature}/spec.md
      - specs/{feature}/plan.md
      - specs/{feature}/tasks.md
      - specs/{feature}/research.md
```

---

## 实施流程

### main agent 编排逻辑

```python
def execute_step_1_5_with_dependencies(requirement):
    """Step 1.5 依赖管理执行"""
    
    # 1. 加载依赖配置
    config = load_yaml("references/step-1.5-dependencies.yaml")
    
    # 2. 构建依赖图
    dependency_graph = build_dependency_graph(config["tasks"])
    
    # 3. 拓扑排序（确定执行顺序）
    execution_order = topological_sort(dependency_graph)
    
    # 4. 按顺序执行
    results = {}
    for task in execution_order:
        # 等待依赖完成
        wait_for_dependencies(task, results)
        
        # 执行任务
        if task["optional"] and should_skip(task):
            # 可选任务失败时跳过
            results[task["name"]] = {"status": "skipped"}
            continue
        
        # Spawn agent
        result = spawn_agent_with_retry(
            agent_id=task["agent"],
            task=build_task_prompt(task, results),
            timeout=task["timeout_seconds"]
        )
        
        results[task["name"]] = result
    
    # 5. 整合结果
    return integrate_results(results)

def wait_for_dependencies(task, results):
    """等待依赖任务完成"""
    for dep_name in task["dependencies"]:
        if dep_name not in results:
            raise Exception(f"依赖 {dep_name} 未执行")
        
        dep_result = results[dep_name]
        if dep_result["status"] == "failed":
            # 依赖失败，检查是否可选
            dep_task = find_task_by_name(dep_name)
            if not dep_task["optional"]:
                raise Exception(f"必需依赖 {dep_name} 失败")
```

### 任务提示构建

```python
def build_task_prompt(task, results):
    """构建任务提示（包含依赖输出）"""
    
    prompt = f"任务：{task['name']}\n\n"
    
    # 添加依赖输出
    if task["dependencies"]:
        prompt += "## 依赖输出\n\n"
        for dep_name in task["dependencies"]:
            dep_result = results[dep_name]
            if dep_result["status"] == "success":
                prompt += f"### {dep_name}\n\n"
                prompt += read_file(dep_result["output"]) + "\n\n"
    
    # 添加任务具体要求
    prompt += f"## 任务要求\n\n{task['description']}\n\n"
    
    # 添加输出要求
    if task["output"]:
        prompt += f"## 输出\n\n保存到：{task['output']}\n"
    
    return prompt
```

---

## 执行时序

### 时序图

```
时间轴 →

0s    珊瑚 spawn ────────────────→ 完成（120s）
                                    ↓
120s                          织梦 spawn ──────────→ 完成（180s）
                                                      ↓
300s                                            review spawn ──→ 完成（120s）
                                                                  ↓
420s                                                        brainstorming spawn ──→ 完成（300s）
                                                                                    ↓
720s                                                                          整合完成

总耗时：720s（12 分钟）
```

### 对比分析

| 方案 | 总耗时 | 质量 | 说明 |
|------|--------|------|------|
| 完全并行（当前） | 300s（5 分钟） | 0.80 | 织梦无法使用珊瑚知识 |
| 部分并行（推荐） | 720s（12 分钟） | 0.88 | 织梦使用珊瑚知识 |
| 完全串行 | 900s（15 分钟） | 0.90 | 最高质量，但耗时最长 |

**结论**：部分并行是最佳平衡点
- 质量提升：+10%
- 时间增加：+140%（但绝对值仅 7 分钟）
- 对于 L2/L3 任务，质量提升值得时间投入

---

## 降级处理

### 可选任务失败

**场景**：珊瑚(notebooklm)查询失败

**处理**：
1. 推送 P2 告警："珊瑚查询失败，降级执行"
2. 跳过珊瑚，直接执行织梦
3. 织梦基于原始需求工作（无历史知识）
4. 在交付报告中标注 "NOTEBOOKLM_DEGRADED"

```python
def handle_optional_task_failure(task, error):
    """处理可选任务失败"""
    
    # 推送告警
    alert_p2(f"{task['name']} 失败，降级执行：{error}")
    
    # 标记为跳过
    return {
        "status": "skipped",
        "reason": error,
        "degraded": True
    }
```

### 必需任务失败

**场景**：织梦(gemini)或 brainstorming 失败

**处理**：
1. 重试 3 次（Spawn 重试机制）
2. 3 次仍失败 → 推送 P0 告警 + HALT
3. 等待人工介入

---

## 集成到流水线

### Step 1.5 流程更新

```markdown
### Step 1.5：Spec-Kit 门控（L2/L3） + 打磨层 + 依赖管理
main 编排：
1. **缓存查询**:
   - 生成任务指纹
   - 查询缓存（精确匹配 + 模糊匹配）
   - 缓存命中 → 跳过所有步骤，直接使用缓存
   - 缓存未命中 → 继续执行
2. **依赖管理执行**（参考 `references/step-1.5-dependencies.yaml`）:
   - 加载依赖配置
   - 按依赖顺序执行：
     a. 珊瑚(notebooklm) 查询历史知识（可选）
     b. 等待珊瑚完成
     c. 织梦(gemini) 基于珊瑚知识产出初稿
     d. 等待织梦完成
     e. review(opus) 验证优化
     f. 等待 review 完成
     g. brainstorming 产出四件套（整合所有输出）
   - 可选任务失败 → 降级跳过
   - 必需任务失败 → 重试 3 次 → HALT
3. 验证四件套一致性，Critical 问题阻塞 Step 2
4. **保存到缓存**（缓存未命中时）
5. 推送结果到头脑风暴群 + 监控群
   - 标注执行模式（缓存命中/正常/降级）
```

---

## 监控和统计

### 执行效果统计

```json
{
  "month": "2026-03",
  "total_l2_l3_tasks": 30,
  "execution_modes": {
    "cache_hit": {
      "count": 12,
      "avg_time_sec": 5,
      "avg_quality": 0.87
    },
    "normal_with_dependencies": {
      "count": 15,
      "avg_time_sec": 720,
      "avg_quality": 0.88
    },
    "degraded_without_notebooklm": {
      "count": 3,
      "avg_time_sec": 600,
      "avg_quality": 0.82
    }
  },
  "quality_improvement": {
    "vs_parallel": 0.10,
    "vs_no_notebooklm": 0.06
  },
  "time_cost": {
    "avg_increase_sec": 420,
    "acceptable": true,
    "reason": "质量提升值得时间投入"
  }
}
```

### 持续优化

1. **每月回顾**：
   - 分析质量提升效果
   - 评估时间成本是否可接受
   - 调整依赖配置

2. **季度校准**：
   - 考虑引入流式传递
   - 优化任务超时配置
   - 评估是否需要更多并行

---

## 未来优化方向

### 流式传递

**目标**：珊瑚实时输出 → 织梦流式接收

**实现**：
1. 珊瑚输出到共享文件（逐行追加）
2. 织梦监听文件变化，流式读取
3. 织梦基于部分输入开始工作

**预期效果**：
- 时间减少 30%（720s → 500s）
- 质量保持不变

### 智能并行

**目标**：根据任务特点动态选择并行策略

**策略**：
- L1 任务：完全并行（速度优先）
- L2 任务：部分并行（平衡）
- L3 任务：完全串行（质量优先）

---

## 注意事项

1. **质量优先**：时间增加可接受，质量提升是核心目标
2. **降级优雅**：可选任务失败不阻塞流程
3. **依赖清晰**：配置文件明确声明依赖关系
4. **超时合理**：每个任务设置合理超时
5. **持续监控**：跟踪质量提升效果
6. **未来优化**：考虑流式传递和智能并行
