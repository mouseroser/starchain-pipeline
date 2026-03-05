# 性能基准对比规范

## 目的

在 Step 3 交叉审查时，对比当前代码与 S1 快照的性能基准，避免功能正确但性能劣化的代码进入生产环境。

## 基准指标

### 1. 响应时间（Response Time）
- **测量对象**：关键函数/API 端点
- **测量方法**：执行 10 次取平均值
- **阈值**：+20%（当前 > 基准 × 1.2 触发 NEEDS_FIX）

### 2. 内存使用（Memory Usage）
- **测量对象**：进程峰值内存
- **测量方法**：执行过程中监控内存峰值
- **阈值**：+30%（当前 > 基准 × 1.3 触发 NEEDS_FIX）

### 3. 依赖调用次数（Dependency Calls）
- **测量对象**：外部服务/数据库调用次数
- **测量方法**：统计执行过程中的调用次数
- **阈值**：+50%（当前 > 基准 × 1.5 触发 NEEDS_FIX）

## 执行流程

### Step 2 完成时（创建 S1 快照）

coding agent 执行性能基准测量：

```bash
# 1. 创建基准记录目录
mkdir -p workspace/benchmarks

# 2. 执行性能测量脚本
python scripts/measure_performance.py --output workspace/benchmarks/S1-baseline.json

# 3. 基准记录格式
{
  "snapshot": "S1",
  "timestamp": 1772716800000,
  "metrics": {
    "response_time": {
      "function_a": 0.125,  // 秒
      "function_b": 0.053
    },
    "memory_usage": {
      "peak_mb": 256
    },
    "dependency_calls": {
      "database": 15,
      "api_external": 3
    }
  }
}
```

### Step 3 审查时

review agent 执行性能对比：

```bash
# 1. 执行当前代码的性能测量
python scripts/measure_performance.py --output workspace/benchmarks/current.json

# 2. 对比基准
python scripts/compare_performance.py \
  --baseline workspace/benchmarks/S1-baseline.json \
  --current workspace/benchmarks/current.json \
  --output workspace/benchmarks/comparison.json

# 3. 对比报告格式
{
  "status": "DEGRADED",  // OK / DEGRADED
  "degradations": [
    {
      "metric": "response_time",
      "function": "function_a",
      "baseline": 0.125,
      "current": 0.162,
      "change_percent": 29.6,
      "threshold": 20,
      "severity": "major"  // minor / major / critical
    }
  ],
  "improvements": [
    {
      "metric": "memory_usage",
      "baseline": 256,
      "current": 220,
      "change_percent": -14.1
    }
  ]
}
```

### 劣化处理规则

| 劣化程度 | 响应时间 | 内存使用 | 依赖调用 | 处理方式 |
|---------|---------|---------|---------|---------|
| **Minor** | +10% ~ +20% | +15% ~ +30% | +25% ~ +50% | PASS_WITH_NOTES（记录到 notes） |
| **Major** | +20% ~ +50% | +30% ~ +60% | +50% ~ +100% | NEEDS_FIX（必须优化） |
| **Critical** | > +50% | > +60% | > +100% | NEEDS_FIX（高优先级） |

### 劣化原因分析

review agent 在发现劣化时，必须分析可能原因：

```json
{
  "degradation": {
    "metric": "response_time",
    "function": "function_a",
    "change_percent": 29.6
  },
  "possible_causes": [
    "新增了 O(n²) 循环",
    "增加了不必要的数据库查询",
    "未使用缓存机制"
  ],
  "suggestions": [
    "优化循环逻辑为 O(n)",
    "合并数据库查询",
    "引入缓存层"
  ]
}
```

## 特殊情况处理

### 1. 无法测量性能
- **原因**：测试环境不可用、依赖服务不可用
- **处理**：跳过性能对比，在 review notes 中标注 "PERF_CHECK_SKIPPED"

### 2. 基准不存在
- **原因**：首次开发、S1 快照未记录基准
- **处理**：跳过性能对比，当前测量结果作为新基准

### 3. 合理的性能劣化
- **场景**：功能扩展导致的合理性能下降
- **处理**：review agent 在 notes 中说明原因，标注 "PERF_DEGRADATION_JUSTIFIED"

## 实施要求

### coding agent（Step 2）
- 在创建 S1 快照时，同步执行性能基准测量
- 将基准记录保存到 `workspace/benchmarks/S1-baseline.json`
- 如果测量失败，记录 warning 但不阻塞流程

### review agent（Step 3）
- 执行当前代码的性能测量
- 对比 S1 基准，生成对比报告
- 根据劣化程度调整 verdict：
  - Minor 劣化 → PASS_WITH_NOTES
  - Major/Critical 劣化 → NEEDS_FIX
- 在 issues JSON 中包含性能劣化详情

### main agent（编排）
- 解析 review 的性能对比报告
- 性能劣化触发 NEEDS_FIX 时，进入 Step 4 修复循环
- 在 Step 4 context bundle 中包含性能对比报告

## 性能测量脚本模板

```python
# scripts/measure_performance.py
import time
import psutil
import json
from pathlib import Path

def measure_response_time(func, iterations=10):
    """测量函数响应时间"""
    times = []
    for _ in range(iterations):
        start = time.time()
        func()
        times.append(time.time() - start)
    return sum(times) / len(times)

def measure_memory_usage(func):
    """测量内存使用峰值"""
    process = psutil.Process()
    before = process.memory_info().rss / 1024 / 1024  # MB
    func()
    after = process.memory_info().rss / 1024 / 1024
    return after

def measure_dependency_calls(func):
    """测量依赖调用次数（需要 mock/instrumentation）"""
    # 实现依赖于具体项目的 instrumentation
    pass

# 使用示例
if __name__ == "__main__":
    # 导入待测试的函数
    from src.main import function_a, function_b
    
    metrics = {
        "response_time": {
            "function_a": measure_response_time(function_a),
            "function_b": measure_response_time(function_b)
        },
        "memory_usage": {
            "peak_mb": measure_memory_usage(function_a)
        }
    }
    
    # 保存基准
    output = Path("workspace/benchmarks/S1-baseline.json")
    output.parent.mkdir(parents=True, exist_ok=True)
    output.write_text(json.dumps(metrics, indent=2))
```

## 注意事项

1. **性能测量不是完整性能测试**：只测量关键路径，不是全面性能分析
2. **环境一致性**：基准测量和当前测量应在相同环境下执行
3. **合理阈值**：阈值可根据项目特点调整（当前为保守值）
4. **不阻塞流程**：性能测量失败不应阻塞流水线，应优雅降级
5. **L1 任务可选**：L1 简单任务可跳过性能对比
