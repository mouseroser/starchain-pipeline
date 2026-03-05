# 异常分类表

## 目的

对 spawn 失败和工具调用异常进行分类，避免无效重试浪费资源，快速定位根因。

---

## 异常分类体系

### 1. 可恢复异常（Recoverable）

**特征**：瞬时故障，重试通常能解决

**处理策略**：重试 3 次
- 第 1 次失败 → 立即重试（相同参数）
- 第 2 次失败 → 等待 10 秒后重试
- 第 3 次仍失败 → 推送 P1 告警到监控群 + 通知晨星，标记该步骤为 BLOCKED

**异常列表**：

| 异常类型 | 错误信息关键词 | 原因 | 示例 |
|---------|--------------|------|------|
| LLM 超时 | `LLM request timed out` | API 响应慢 | Gemini 生图超时 |
| API 限流 | `503`, `rate limit`, `too many requests` | 请求频率过高 | OpenAI API 限流 |
| 网络错误 | `connection error`, `network timeout`, `ECONNREFUSED` | 网络抖动 | 临时网络中断 |
| 临时服务不可用 | `502`, `504`, `service unavailable` | 上游服务重启 | Gateway 临时不可用 |

---

### 2. 不可恢复异常（Unrecoverable）

**特征**：配置或权限问题，重试无法解决

**处理策略**：立即 HALT
- 推送 P0 告警到监控群 + 通知晨星
- 输出详细错误信息和修复建议
- 标记流水线状态为 BLOCKED
- 等待人工介入修复

**异常列表**：

| 异常类型 | 错误信息关键词 | 原因 | 修复建议 |
|---------|--------------|------|---------|
| 认证失败 | `auth_missing`, `auth_expired`, `authentication failed`, `401` | API key 缺失或过期 | 检查 openclaw.json 中的 API key 配置 |
| 配置错误 | `config error`, `missing required file`, `invalid config` | 配置文件缺失或格式错误 | 检查 openclaw.json / settings.json |
| 语法错误 | `syntax error`, `parse error`, `invalid JSON` | 流水线合约或配置文件语法错误 | 运行 `openclaw doctor --fix` |
| 权限不足 | `permission denied`, `403`, `access denied` | 文件或 API 权限不足 | 检查文件权限或 API 授权 |
| 依赖缺失 | `module not found`, `command not found`, `ENOENT` | 必要的依赖未安装 | 安装缺失的依赖（npm/pip/brew） |

---

### 3. 降级异常（Degradable）

**特征**：可选功能失败，不影响核心流程

**处理策略**：优雅降级
- 推送 P2 告警到监控群（Warning 级别）
- 跳过该步骤，使用降级方案
- 继续流水线执行
- 在交付报告中标注降级项

**异常列表**：

| 异常类型 | 错误信息关键词 | 降级方案 | 影响 |
|---------|--------------|---------|------|
| NotebookLM 不可用 | `nlm-gateway.sh` 失败, `auth_missing` | 跳过珊瑚查询，依赖模型权重 | Step 1.5/6 质量略降 |
| Gemini 不可用 | `gemini` spawn 失败, `503` | 跳过织梦加速，直接用 brainstorming | Step 1.5 效率降低 30% |
| 性能测量失败 | `measure_performance.py` 失败 | 跳过性能对比，标注 PERF_CHECK_SKIPPED | Step 3 缺少性能把关 |
| 文档生成失败 | `docs` agent 失败 | 跳过 Step 6，交付时标注 DOCS_SKIPPED | 缺少文档更新 |
| 冒烟测试部分失败 | 非核心路径失败 | 标注 Warning，继续 Step 3 | 可能在 Step 5 发现问题 |

---

## 实施规范

### main agent（编排中心）

在 Spawn 重试机制中增加异常分类逻辑：

```python
# 伪代码
def spawn_with_retry(agent_id, task, max_retries=3):
    for attempt in range(1, max_retries + 1):
        result = sessions_spawn(agentId=agent_id, task=task)
        
        if result.status == "accepted":
            # 等待 announce 回调
            return result
        
        # 异常分类
        error_type = classify_exception(result.error)
        
        if error_type == "UNRECOVERABLE":
            # 立即 HALT
            alert_p0(f"{agent_id} 不可恢复异常: {result.error}")
            notify_chenxing(f"流水线 BLOCKED: {result.error}")
            return {"status": "BLOCKED", "reason": result.error}
        
        elif error_type == "DEGRADABLE":
            # 优雅降级
            alert_p2(f"{agent_id} 降级: {result.error}")
            return {"status": "DEGRADED", "reason": result.error}
        
        elif error_type == "RECOVERABLE":
            # 重试
            if attempt == 2:
                time.sleep(10)  # 第 2 次重试前等待
            if attempt == max_retries:
                # 3 次都失败
                alert_p1(f"{agent_id} 重试 3 次失败: {result.error}")
                notify_chenxing(f"{agent_id} BLOCKED")
                return {"status": "BLOCKED", "reason": result.error}
            continue

def classify_exception(error_msg):
    """异常分类"""
    error_lower = error_msg.lower()
    
    # 不可恢复异常
    unrecoverable_keywords = [
        "auth_missing", "auth_expired", "authentication failed", "401",
        "config error", "missing required file", "invalid config",
        "syntax error", "parse error", "invalid json",
        "permission denied", "403", "access denied",
        "module not found", "command not found", "enoent"
    ]
    if any(kw in error_lower for kw in unrecoverable_keywords):
        return "UNRECOVERABLE"
    
    # 降级异常
    degradable_keywords = [
        "nlm-gateway", "珊瑚", "notebooklm",
        "gemini", "织梦",
        "measure_performance", "perf_check",
        "docs agent", "documentation"
    ]
    if any(kw in error_lower for kw in degradable_keywords):
        return "DEGRADABLE"
    
    # 默认为可恢复异常
    return "RECOVERABLE"
```

### 告警推送规范

```python
def alert_p0(message):
    """P0 告警：立即处理"""
    message_tool(
        action="send",
        channel="telegram",
        target="-5131273722",  # 监控群
        message=f"🚨 P0 告警\n{message}"
    )

def alert_p1(message):
    """P1 告警：1小时内处理"""
    message_tool(
        action="send",
        channel="telegram",
        target="-5131273722",
        message=f"⚠️ P1 告警\n{message}"
    )

def alert_p2(message):
    """P2 告警：24小时内处理"""
    message_tool(
        action="send",
        channel="telegram",
        target="-5131273722",
        message=f"ℹ️ P2 告警\n{message}"
    )

def notify_chenxing(message):
    """通知晨星"""
    message_tool(
        action="send",
        channel="telegram",
        target="1099011886",
        message=message
    )
```

---

## 降级方案详细说明

### 1. NotebookLM 降级

**触发条件**：`nlm-gateway.sh` 认证失败或 CLI 错误

**降级流程**：
1. 推送 P2 告警："珊瑚(notebooklm)不可用，跳过历史知识查询"
2. 跳过珊瑚 spawn
3. 织梦和 brainstorming 直接基于原始需求工作
4. 在交付报告中标注："NOTEBOOKLM_DEGRADED"

**影响评估**：
- Step 1.5 质量略降（缺少历史知识）
- Step 6 文档质量略降（缺少模板参考）
- 不影响核心开发流程

---

### 2. Gemini 降级

**触发条件**：gemini agent spawn 失败（503/超时）

**降级流程**：
1. 推送 P2 告警："织梦(gemini)不可用，跳过研究加速"
2. 跳过织梦 spawn
3. brainstorming 直接产出四件套（无初稿参考）
4. 在交付报告中标注："GEMINI_DEGRADED"

**影响评估**：
- Step 1.5 效率降低 30-40%
- 质量略降（缺少初稿打磨）
- 不影响核心开发流程

---

### 3. 性能测量降级

**触发条件**：`measure_performance.py` 执行失败

**降级流程**：
1. 推送 P2 告警："性能测量失败，跳过性能对比"
2. review 在 Step 3 跳过性能基准对比
3. 在 verdict JSON 中标注："PERF_CHECK_SKIPPED"
4. 在交付报告中标注："PERFORMANCE_CHECK_DEGRADED"

**影响评估**：
- Step 3 缺少性能把关
- 可能在生产环境发现性能劣化
- 不影响功能正确性

---

## 监控和统计

### 异常统计

在 `workspace/exception-stats.json` 记录异常统计：

```json
{
  "period": "2026-03-01 to 2026-03-31",
  "total_exceptions": 45,
  "by_type": {
    "RECOVERABLE": 32,
    "UNRECOVERABLE": 5,
    "DEGRADABLE": 8
  },
  "by_agent": {
    "gemini": 12,
    "notebooklm": 8,
    "coding": 15,
    "review": 5,
    "test": 5
  },
  "top_errors": [
    {"error": "LLM request timed out", "count": 18},
    {"error": "nlm-gateway auth_missing", "count": 8},
    {"error": "503 rate limit", "count": 7}
  ]
}
```

### 定期回顾

每月回顾异常统计：
1. 识别高频异常
2. 优化重试策略
3. 改进降级方案
4. 更新异常分类表

---

## 注意事项

1. **保守分类**：不确定时默认为 RECOVERABLE，避免误判为 UNRECOVERABLE
2. **快速失败**：UNRECOVERABLE 异常立即 HALT，不浪费资源
3. **优雅降级**：DEGRADABLE 异常不阻塞流程，但必须记录
4. **持续优化**：根据异常统计持续优化分类规则
5. **人工介入**：UNRECOVERABLE 异常需要人工修复配置后重启流水线
