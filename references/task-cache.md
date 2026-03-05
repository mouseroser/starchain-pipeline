# 任务指纹缓存机制

## 目的

对相似任务缓存研究结果和文档模板，避免重复工作，节省 20-30% 成本。

---

## 任务指纹生成

### 指纹算法

```python
import hashlib
import json

def generate_task_fingerprint(requirement, files, dependencies):
    """生成任务指纹"""
    
    # 1. 提取关键词
    keywords = extract_keywords(requirement)
    keywords.sort()  # 排序保证一致性
    
    # 2. 提取文件路径（归一化）
    file_paths = [normalize_path(f) for f in files]
    file_paths.sort()
    
    # 3. 提取依赖列表
    deps = sorted(dependencies)
    
    # 4. 组合为字符串
    fingerprint_data = {
        "keywords": keywords,
        "files": file_paths,
        "dependencies": deps
    }
    
    # 5. 计算哈希
    data_str = json.dumps(fingerprint_data, sort_keys=True)
    fingerprint = hashlib.sha256(data_str.encode()).hexdigest()[:16]
    
    return fingerprint, fingerprint_data

# 示例
requirement = "添加用户登录 API"
files = ["src/auth.py", "src/models/user.py", "tests/test_auth.py"]
dependencies = ["flask", "sqlalchemy", "jwt"]

fingerprint, data = generate_task_fingerprint(requirement, files, dependencies)
# 输出：fingerprint = "a3f5c8d2e1b4f7a9"
```

### 关键词提取

```python
def extract_keywords(requirement):
    """提取需求关键词"""
    
    # 1. 分词
    words = tokenize(requirement)
    
    # 2. 过滤停用词
    stopwords = ["的", "是", "在", "和", "与", "或", "等"]
    words = [w for w in words if w not in stopwords]
    
    # 3. 提取技术关键词
    tech_keywords = []
    tech_patterns = [
        r"API", r"数据库", r"缓存", r"认证", r"授权",
        r"登录", r"注册", r"查询", r"优化", r"重构"
    ]
    for word in words:
        for pattern in tech_patterns:
            if re.search(pattern, word, re.IGNORECASE):
                tech_keywords.append(word.lower())
    
    # 4. 去重并排序
    return sorted(set(tech_keywords))
```

---

## 缓存结构

### 缓存目录结构

```
workspace/cache/
├── index.json                    # 缓存索引
├── a3f5c8d2e1b4f7a9/             # 任务指纹目录
│   ├── metadata.json             # 元数据
│   ├── step-1.5-research.md      # Step 1.5 研究结果
│   ├── step-6-doc-template.md    # Step 6 文档模板
│   └── stats.json                # 使用统计
└── b7e2f9a1c4d8e3f6/
    └── ...
```

### 缓存索引格式

`workspace/cache/index.json`：

```json
{
  "caches": [
    {
      "fingerprint": "a3f5c8d2e1b4f7a9",
      "keywords": ["api", "登录", "认证"],
      "files": ["src/auth.py", "src/models/user.py"],
      "dependencies": ["flask", "jwt", "sqlalchemy"],
      "created_at": 1772716800000,
      "last_used_at": 1772803200000,
      "use_count": 5,
      "expires_at": 1773321600000,
      "cached_steps": ["step_1.5", "step_6"]
    },
    {
      "fingerprint": "b7e2f9a1c4d8e3f6",
      "keywords": ["数据库", "查询", "优化"],
      "files": ["src/db/query.py"],
      "dependencies": ["sqlalchemy"],
      "created_at": 1772720400000,
      "last_used_at": 1772720400000,
      "use_count": 1,
      "expires_at": 1773325200000,
      "cached_steps": ["step_1.5"]
    }
  ],
  "stats": {
    "total_caches": 2,
    "total_hits": 6,
    "total_misses": 15,
    "hit_rate": 0.286,
    "tokens_saved": 180000
  }
}
```

### 缓存元数据格式

`workspace/cache/{fingerprint}/metadata.json`：

```json
{
  "fingerprint": "a3f5c8d2e1b4f7a9",
  "task_id": "task-20260305-001",
  "requirement": "添加用户登录 API",
  "level": "L2",
  "keywords": ["api", "登录", "认证"],
  "files": ["src/auth.py", "src/models/user.py", "tests/test_auth.py"],
  "dependencies": ["flask", "sqlalchemy", "jwt"],
  "created_at": 1772716800000,
  "created_by": "main",
  "cached_steps": {
    "step_1.5": {
      "file": "step-1.5-research.md",
      "tokens": 12000,
      "quality_score": 0.85
    },
    "step_6": {
      "file": "step-6-doc-template.md",
      "tokens": 8000,
      "quality_score": 0.90
    }
  },
  "usage": {
    "use_count": 5,
    "last_used_at": 1772803200000,
    "tasks_used": [
      "task-20260305-002",
      "task-20260306-001",
      "task-20260307-003"
    ]
  },
  "expiration": {
    "expires_at": 1773321600000,
    "ttl_days": 7,
    "invalidation_triggers": [
      "dependency_change",
      "file_structure_change"
    ]
  }
}
```

---

## 缓存查询

### 查询流程

```python
def query_cache(requirement, files, dependencies):
    """查询缓存"""
    
    # 1. 生成指纹
    fingerprint, data = generate_task_fingerprint(requirement, files, dependencies)
    
    # 2. 精确匹配
    exact_match = find_exact_match(fingerprint)
    if exact_match and not is_expired(exact_match):
        return {
            "hit": True,
            "match_type": "exact",
            "fingerprint": fingerprint,
            "cache": exact_match
        }
    
    # 3. 模糊匹配（相似度 > 0.8）
    similar_matches = find_similar_matches(data, threshold=0.8)
    if similar_matches:
        best_match = max(similar_matches, key=lambda x: x["similarity"])
        if not is_expired(best_match):
            return {
                "hit": True,
                "match_type": "similar",
                "similarity": best_match["similarity"],
                "fingerprint": best_match["fingerprint"],
                "cache": best_match
            }
    
    # 4. 缓存未命中
    return {
        "hit": False,
        "fingerprint": fingerprint
    }
```

### 相似度计算

```python
def calculate_similarity(data1, data2):
    """计算任务相似度"""
    
    # 1. 关键词相似度（Jaccard）
    keywords1 = set(data1["keywords"])
    keywords2 = set(data2["keywords"])
    keyword_sim = len(keywords1 & keywords2) / len(keywords1 | keywords2)
    
    # 2. 文件路径相似度
    files1 = set(data1["files"])
    files2 = set(data2["files"])
    file_sim = len(files1 & files2) / len(files1 | files2)
    
    # 3. 依赖相似度
    deps1 = set(data1["dependencies"])
    deps2 = set(data2["dependencies"])
    dep_sim = len(deps1 & deps2) / len(deps1 | deps2) if (deps1 | deps2) else 1.0
    
    # 4. 加权平均
    similarity = (
        keyword_sim * 0.5 +
        file_sim * 0.3 +
        dep_sim * 0.2
    )
    
    return similarity
```

---

## 缓存使用

### Step 1.5 缓存使用

```python
def step_1_5_with_cache(requirement, files, dependencies):
    """Step 1.5 使用缓存"""
    
    # 1. 查询缓存
    cache_result = query_cache(requirement, files, dependencies)
    
    if cache_result["hit"]:
        # 缓存命中
        fingerprint = cache_result["fingerprint"]
        cache_dir = f"workspace/cache/{fingerprint}"
        
        # 读取缓存的研究结果
        research_md = read_file(f"{cache_dir}/step-1.5-research.md")
        
        # 更新使用统计
        update_cache_usage(fingerprint)
        
        # 推送通知
        notify_cache_hit(fingerprint, cache_result["match_type"])
        
        # 跳过珊瑚和织梦，直接使用缓存
        return {
            "status": "cache_hit",
            "research": research_md,
            "tokens_saved": cache_result["cache"]["cached_steps"]["step_1.5"]["tokens"]
        }
    else:
        # 缓存未命中，正常执行
        result = execute_step_1_5_normal(requirement)
        
        # 保存到缓存
        save_to_cache(cache_result["fingerprint"], "step_1.5", result)
        
        return {
            "status": "cache_miss",
            "research": result["research"]
        }
```

### Step 6 缓存使用

```python
def step_6_with_cache(requirement, files, dependencies):
    """Step 6 使用缓存"""
    
    # 1. 查询缓存
    cache_result = query_cache(requirement, files, dependencies)
    
    if cache_result["hit"]:
        # 缓存命中
        fingerprint = cache_result["fingerprint"]
        cache_dir = f"workspace/cache/{fingerprint}"
        
        # 读取缓存的文档模板
        doc_template = read_file(f"{cache_dir}/step-6-doc-template.md")
        
        # 更新使用统计
        update_cache_usage(fingerprint)
        
        # 推送通知
        notify_cache_hit(fingerprint, cache_result["match_type"])
        
        # 使用模板生成文档（仍需 docs agent 填充具体内容）
        return {
            "status": "cache_hit",
            "template": doc_template,
            "tokens_saved": cache_result["cache"]["cached_steps"]["step_6"]["tokens"]
        }
    else:
        # 缓存未命中，正常执行
        result = execute_step_6_normal(requirement)
        
        # 保存到缓存
        save_to_cache(cache_result["fingerprint"], "step_6", result)
        
        return {
            "status": "cache_miss",
            "template": result["template"]
        }
```

---

## 缓存失效

### 失效策略

#### 1. 时间失效（TTL）

**默认 TTL**：7 天

```python
def is_expired(cache):
    """检查缓存是否过期"""
    now = time.time() * 1000  # 毫秒
    return now > cache["expires_at"]
```

#### 2. 依赖变更失效

**触发条件**：
- 依赖版本升级
- 新增/删除依赖
- 依赖配置变更

```python
def check_dependency_invalidation(cache, current_dependencies):
    """检查依赖是否变更"""
    cached_deps = set(cache["dependencies"])
    current_deps = set(current_dependencies)
    
    # 依赖不一致 → 失效
    if cached_deps != current_deps:
        invalidate_cache(cache["fingerprint"], reason="dependency_change")
        return True
    
    return False
```

#### 3. 文件结构变更失效

**触发条件**：
- 文件路径变更
- 文件删除
- 目录结构重组

```python
def check_file_structure_invalidation(cache, current_files):
    """检查文件结构是否变更"""
    cached_files = set(cache["files"])
    current_files = set(current_files)
    
    # 文件结构变化超过 30% → 失效
    overlap = len(cached_files & current_files)
    similarity = overlap / len(cached_files | current_files)
    
    if similarity < 0.7:
        invalidate_cache(cache["fingerprint"], reason="file_structure_change")
        return True
    
    return False
```

#### 4. 手动失效

```python
def manual_invalidate(fingerprint, reason):
    """手动失效缓存"""
    cache_dir = f"workspace/cache/{fingerprint}"
    
    # 标记为失效
    metadata = load_json(f"{cache_dir}/metadata.json")
    metadata["invalidated"] = True
    metadata["invalidation_reason"] = reason
    metadata["invalidated_at"] = time.time() * 1000
    
    save_json(f"{cache_dir}/metadata.json", metadata)
    
    # 从索引中移除
    remove_from_index(fingerprint)
```

---

## 缓存维护

### 定期清理

**清理策略**：
- 每周清理过期缓存
- 每月清理低使用率缓存（use_count < 2）
- 缓存总大小超过 1GB 时清理最旧的缓存

```python
def cleanup_cache():
    """清理缓存"""
    
    index = load_json("workspace/cache/index.json")
    now = time.time() * 1000
    
    removed = []
    
    for cache in index["caches"]:
        # 1. 过期缓存
        if now > cache["expires_at"]:
            remove_cache(cache["fingerprint"])
            removed.append(cache["fingerprint"])
            continue
        
        # 2. 低使用率缓存（创建超过 30 天且使用次数 < 2）
        age_days = (now - cache["created_at"]) / (1000 * 60 * 60 * 24)
        if age_days > 30 and cache["use_count"] < 2:
            remove_cache(cache["fingerprint"])
            removed.append(cache["fingerprint"])
            continue
    
    # 3. 检查总大小
    total_size = get_cache_total_size()
    if total_size > 1024 * 1024 * 1024:  # 1GB
        # 按最后使用时间排序，删除最旧的
        sorted_caches = sorted(index["caches"], key=lambda x: x["last_used_at"])
        for cache in sorted_caches:
            if cache["fingerprint"] not in removed:
                remove_cache(cache["fingerprint"])
                removed.append(cache["fingerprint"])
                total_size = get_cache_total_size()
                if total_size < 800 * 1024 * 1024:  # 降到 800MB
                    break
    
    return removed
```

---

## 集成到流水线

### Step 1.5 流程更新

```markdown
### Step 1.5：Spec-Kit 门控（L2/L3） + 打磨层 + 缓存
main 编排：
1. **缓存查询**（参考 `references/task-cache.md`）:
   - 生成任务指纹
   - 查询缓存（精确匹配 + 模糊匹配）
   - 缓存命中 → 跳过珊瑚和织梦，直接使用缓存
   - 缓存未命中 → 正常执行
2. **并行执行知识查询和初步分析**（缓存未命中时）:
   - 同时 spawn 珊瑚(notebooklm) 和 织梦(gemini)
   - ...（原流程）
3. **保存到缓存**（缓存未命中时）:
   - 保存研究结果到 workspace/cache/{fingerprint}/
   - 更新缓存索引
4. 推送结果到头脑风暴群 + 监控群
   - 缓存命中时标注 "CACHE_HIT"
```

### Step 6 流程更新

```markdown
### Step 6：文档（L1 跳过） + 缓存
main 编排：
1. **缓存查询**:
   - 查询文档模板缓存
   - 缓存命中 → 使用模板，跳过珊瑚和织梦
   - 缓存未命中 → 正常执行
2. **珊瑚知识查询**（缓存未命中时，可选）:
   - ...（原流程）
3. **织梦加速**（缓存未命中时）:
   - ...（原流程）
4. **docs 生成**:
   - 使用缓存模板或新生成的模板
   - 填充具体内容
5. **保存到缓存**（缓存未命中时）:
   - 保存文档模板到缓存
6. 推送文档群 + 监控群
```

---

## 监控和统计

### 缓存效果统计

```json
{
  "month": "2026-03",
  "total_tasks": 50,
  "cache_stats": {
    "step_1.5": {
      "queries": 30,
      "hits": 12,
      "hit_rate": 0.40,
      "tokens_saved": 144000,
      "time_saved_min": 60
    },
    "step_6": {
      "queries": 30,
      "hits": 15,
      "hit_rate": 0.50,
      "tokens_saved": 120000,
      "time_saved_min": 45
    }
  },
  "total_savings": {
    "tokens": 264000,
    "time_min": 105,
    "cost_reduction": 0.26
  },
  "cache_quality": {
    "exact_matches": 18,
    "similar_matches": 9,
    "false_positives": 2,
    "accuracy": 0.93
  }
}
```

### 持续优化

1. **每月回顾**：
   - 分析缓存命中率
   - 识别高频任务模式
   - 调整相似度阈值

2. **季度校准**：
   - 评估缓存质量
   - 优化指纹算法
   - 调整 TTL 策略

---

## 注意事项

1. **缓存不是银弹**：只缓存研究结果和模板，不缓存代码
2. **失效策略保守**：宁可多失效，不可用过期缓存
3. **相似度阈值**：0.8 是保守值，可根据实际调整
4. **定期清理**：避免缓存无限增长
5. **质量监控**：跟踪缓存导致的质量问题
6. **透明可追溯**：缓存命中必须记录，支持审计
