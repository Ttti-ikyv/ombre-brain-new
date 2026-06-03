# Ombre Brain 用户全流程行为规格书

> 版本：基于 server.py / bucket_manager.py / decay_engine.py / dehydrator.py / embedding_engine.py / CLAUDE_PROMPT.md / config.example.yaml

---

## 一、系统角色说明

### 1.1 参与方总览

| 角色 | 实体 | 职责边界 |
|------|------|---------|
| **用户** | 人类 | 发起对话，提供原始内容；可直接访问 Dashboard Web UI |
| **Claude（模型端）** | LLM（如 Claude 3.x）| 理解语义、决策何时调用工具、用自然语言回应用户；不直接操作文件 |
| **OB 服务端** | `server.py` + 各模块 | 接收 MCP 工具调用，执行持久化、搜索、衰减；对 Claude 不透明 |

### 1.2 Claude 端职责边界
- **必须做**：每次新对话第一步无参调用 `breath()`；对话内容有记忆价值时主动调用 `hold` / `grow`
- **不做**：不直接读写 `.md` 文件；不执行衰减计算；不操作 SQLite
- **决策权**：Claude 决定是否存、存哪些、何时 resolve；OB 决定如何存（合并/新建）

### 1.3 OB 服务端内部模块职责

| 模块 | 核心职责 |
|------|---------|
| `server.py` | 注册 MCP 工具（`breath/hold/grow/trace/pulse/dream`）；路由 Dashboard HTTP 请求；`_merge_or_create()` 合并逻辑中枢 |
| `bucket_manager.py` | 桶 CRUD；多维搜索（fuzzy + embedding 双通道）；`touch()` 激活刷新；`_time_ripple()` 时间波纹 |
| `dehydrator.py` | `analyze()` 自动打标；`merge()` 内容融合；`digest()` 日记拆分；`dehydrate()` 内容压缩 |
| `embedding_engine.py` | `generate_and_store()` 生成向量并存 SQLite；`search_similar()` 余弦相似度检索 |
| `decay_engine.py` | `calculate_score()` 衰减分计算；`run_decay_cycle()` 周期扫描归档；后台定时循环 |
| `utils.py` | 配置加载；路径安全校验；ID 生成；token 估算 |

---

## 二、场景全流程

---

### 场景 1：新对话开始（冷启动，无历史记忆）

**用户操作**：打开新对话窗口，说第一句话

**Claude 行为**：在任何回复之前，先调用 `breath()`（无参）

**OB 工具调用**：
```
breath(query="", max_tokens=10000, domain="", valence=-1, arousal=-1, max_results=20, importance_min=-1)
```

**系统内部发生什么**：
1. `decay_engine.ensure_started()` — 懒加载启动后台衰减循环（若未运行）
2. 进入"浮现模式"（`not query or not query.strip()`）
3. `bucket_mgr.list_all(include_archive=False)` — 遍历 `permanent/` + `dynamic/` + `feel/` 目录，加载所有 `.md` 文件的 frontmatter + 正文
4. 筛选钉选桶（`pinned=True` 或 `protected=True`）
5. 筛选未解决桶（`resolved=False`，排除 `permanent/feel/pinned`）
6. **冷启动检测**：找 `activation_count==0 && importance>=8` 的桶，最多取 2 个插入排序最前（**决策：`create()` 初始化应为 0，区分"创建"与"被主动召回"，见 B-04**）
7. 按 `decay_engine.calculate_score(metadata)` 降序排列剩余未解决桶
8. 对 top-20 以外随机洗牌（top-1 固定，2~20 随机）
9. 截断到 `max_results` 条
10. 对每个桶调用 `dehydrator.dehydrate(strip_wikilinks(content), clean_meta)` 压缩摘要
11. 按 `max_tokens` 预算截断输出

**返回结果**：
- 无记忆时：`"权重池平静，没有需要处理的记忆。"`
- 有记忆时：`"=== 核心准则 ===\n📌 ...\n\n=== 浮现记忆 ===\n[权重:X.XX] [bucket_id:xxx] ..."`

**注意**：浮现模式**不调用** `touch()`，不重置衰减计时器

---

### 场景 2：新对话开始（有历史记忆，breath 自动浮现）

（与场景 1 相同流程，区别在于桶文件已存在）

**Claude 行为（完整对话启动序列，来自 CLAUDE_PROMPT.md）**：
```
1. breath()               — 浮现未解决记忆
2. dream()                — 消化最近记忆，有沉淀写 feel
3. breath(domain="feel")  — 读取之前的 feel
4. 开始和用户说话
```

**`breath(domain="feel")` 内部流程**：
1. 检测到 `domain.strip().lower() == "feel"` → 进入 feel 专用通道
2. `bucket_mgr.list_all()` 过滤 `type=="feel"` 的桶
3. 按 `created` 降序排列
4. 按 `max_tokens` 截断，不压缩（直接展示原文）
5. 返回：`"=== 你留下的 feel ===\n[时间] [bucket_id:xxx]\n内容..."`

---

### 场景 3：用户说了一件事，Claude 决定存入记忆（hold）

**用户操作**：例如"我刚刚拿到了实习 offer，有点激动"

**Claude 行为**：判断值得记忆，调用：
```python
hold(content="用户拿到实习 offer，情绪激动", importance=7)
```

**OB 工具调用**：`hold(content, tags="", importance=7, pinned=False, feel=False, source_bucket="", valence=-1, arousal=-1)`

**系统内部发生什么**：

1. `decay_engine.ensure_started()`
2. 输入校验：`content.strip()` 非空
3. `importance = max(1, min(10, 7))` = 7
4. `extra_tags = []`（未传 tags）
5. **自动打标**：`dehydrator.analyze(content)` → 调用 `_api_analyze()` → LLM 返回 JSON
   - 返回示例：`{"domain": ["成长", "求职"], "valence": 0.8, "arousal": 0.7, "tags": ["实习", "offer", "激动", ...], "suggested_name": "实习offer获得"}`
   - 失败时降级：`{"domain": ["未分类"], "valence": 0.5, "arousal": 0.3, "tags": [], "suggested_name": ""}`
6. 合并 `auto_tags + extra_tags` 去重
7. **合并检测**：`_merge_or_create(content, tags, importance=7, domain, valence, arousal, name)`
   - `bucket_mgr.search(content, limit=1, domain_filter=domain)` — 搜索最相似的桶
   - 若最高分 > `config["merge_threshold"]`（默认 75）且该桶非 pinned/protected：
     - `dehydrator.merge(old_content, new_content)` → `_api_merge()` → LLM 融合
     - `bucket_mgr.update(bucket_id, content=merged, tags=union, importance=max, domain=union, valence=avg, arousal=avg)`
     - `embedding_engine.generate_and_store(bucket_id, merged_content)` 更新向量
     - 返回 `(bucket_name, True)`
   - 否则：
     - `bucket_mgr.create(content, tags, importance=7, domain, valence, arousal, name)` → 写 `.md` 文件到 `dynamic/<主题域>/` 目录
     - `embedding_engine.generate_and_store(bucket_id, content)` 生成并存储向量
     - 返回 `(bucket_id, False)`

**返回结果**：
- 新建：`"新建→实习offer获得 成长,求职"`
- 合并：`"合并→求职经历 成长,求职"`

**bucket_mgr.create() 详情**：
- `generate_bucket_id()` → `uuid4().hex[:12]`
- `sanitize_name(name)` → 正则清洗，最长 80 字符
- 写 YAML frontmatter + 正文到 `safe_path(domain_dir, f"{name}_{id}.md")`
- frontmatter 字段：`id, name, tags, domain, valence, arousal, importance, type, created, last_active, activation_count=0`（**决策：初始为 0，`touch()` 首次被召回后变为 1**）

---

### 场景 4：用户说了一段长日记，Claude 整理存入（grow）

**用户操作**：发送一大段混合内容，如"今天去医院体检，结果还好；晚上和朋友吃饭聊了很多；最近有点焦虑..."

**Claude 行为**：
```python
grow(content="今天去医院体检，结果还好；晚上和朋友吃饭聊了很多；最近有点焦虑...")
```

**系统内部发生什么**：

1. `decay_engine.ensure_started()`
2. 内容长度检查：`len(content.strip()) < 30` → 若短于 30 字符走**快速路径**（`dehydrator.analyze()` + `_merge_or_create()`，跳过 digest）
3. **日记拆分**（正常路径）：`dehydrator.digest(content)` → `_api_digest()` → LLM 调用 `DIGEST_PROMPT`
   - LLM 返回 JSON 数组，每项含：`name, content, domain, valence, arousal, tags, importance`
   - `_parse_digest()` 安全解析，校验 valence/arousal 范围
4. 对每个 `item` 调用 `_merge_or_create(item["content"], item["tags"], item["importance"], item["domain"], item["valence"], item["arousal"], item["name"])`
   - 每项独立走合并或新建逻辑（同场景 3）
   - 单条失败不影响其他条（`try/except` 隔离）

**返回结果**：
```
3条|新2合1
📝体检结果
📌朋友聚餐
📎近期焦虑情绪
```

---

### 场景 5：用户想找某段记忆（breath 带 query 检索）

**用户操作**：例如"还记得我之前说过关于实习的事吗"

**Claude 行为**：
```python
breath(query="实习", domain="成长", valence=0.7, arousal=0.5)
```

**系统内部发生什么**：

1. `decay_engine.ensure_started()`
2. 检测到 `query` 非空，进入**检索模式**
3. 解析 `domain_filter = ["成长"]`，`q_valence=0.7`，`q_arousal=0.5`
4. **关键词检索**：`bucket_mgr.search(query, limit=20, domain_filter, q_valence, q_arousal)`
   - **Layer 1**：domain 预筛 → 仅保留 domain 包含"成长"的桶；若为空则回退全量
   - **Layer 1.5**（embedding 已开启时）：`embedding_engine.search_similar(query, top_k=50)` → 用 embedding 候选集替换/缩小精排范围
   - **Layer 2**：多维加权精排：
     - `_calc_topic_score()`: `fuzz.partial_ratio(query, name)×3 + domain×2.5 + tags×2 + body×1`，归一化 0~1
     - `_calc_emotion_score()`: `1 - √((v差²+a差²)/2)`，0~1
     - `_calc_time_score()`: `e^(-0.02×days_since_last_active)`，0~1
     - `importance_score`: `importance / 10`
     - `total = topic×4 + emotion×2 + time×1.5 + importance×1`，归一化到 0~100
     - 过滤 `score >= fuzzy_threshold`（默认 50）
     - 通过阈值后，`resolved` 桶仅在排序时降权 ×0.3（不影响是否被检出）
     - 返回最多 `limit` 条
5. 排除 pinned/protected 桶（它们在浮现模式展示）
6. **向量补充通道**（server.py 额外层）：`embedding_engine.search_similar(query, top_k=20)` → 相似度 > 0.5 的桶补充到结果集（标记 `vector_match=True`）
7. 对每个结果：
   - 记忆重构：若传了 `q_valence`，展示层 valence 做微调：`shift = (q_valence - 0.5) × 0.2`，最大 ±0.1
   - `dehydrator.dehydrate(strip_wikilinks(content), clean_meta)` 压缩摘要
   - `bucket_mgr.touch(bucket_id)` — 刷新 `last_active` + `activation_count += 1` + 触发 `_time_ripple()`（对 48h 内创建的邻近桶 activation_count + 0.3，最多 5 个桶）
8. **随机漂流**：若检索结果 < 3 且 `random.random() < 0.4`，随机从 `decay_score < 2.0` 的旧桶里取 1~3 条，标注 `[surface_type: random]`

**返回结果**：
```
[bucket_id:abc123] [重要度:7] [主题:成长] 实习offer获得：...
[语义关联] [bucket_id:def456] 求职经历...
--- 忽然想起来 ---
[surface_type: random] 某段旧记忆...
```

---

### 场景 6：用户想查看所有记忆状态（pulse）

**用户操作**："帮我看看你现在都记得什么"

**Claude 行为**：
```python
pulse(include_archive=False)
```

**系统内部发生什么**：

1. `bucket_mgr.get_stats()` — 遍历三个目录，统计文件数量和 KB 大小
2. `bucket_mgr.list_all(include_archive=False)` — 加载全部桶
3. 对每个桶：`decay_engine.calculate_score(metadata)` 计算当前权重分
4. 按类型/状态分配图标：📌钉选 / 📦permanent / 🫧feel / 🗄️archived / ✅resolved / 💭普通
5. 拼接每桶摘要行：`名称 bucket_id 主题 情感坐标 重要度 权重 标签`

**返回结果**：
```
=== Ombre Brain 记忆系统 ===
固化记忆桶: 2 个
动态记忆桶: 15 个
归档记忆桶: 3 个
总存储大小: 48.3 KB
衰减引擎: 运行中

=== 记忆列表 ===
📌 [核心原则] bucket_id:abc123 主题:内心 情感:V0.8/A0.5 ...
💭 [实习offer获得] bucket_id:def456 主题:成长 情感:V0.8/A0.7 ...
```

---

### 场景 7：用户想修改/标记已解决/删除某条记忆（trace）

#### 7a 标记已解决

**Claude 行为**：
```python
trace(bucket_id="abc123", resolved=1)
```

**系统内部**：
1. `resolved in (0, 1)` → `updates["resolved"] = True`
2. `bucket_mgr.update("abc123", resolved=True)` → 读取 `.md` 文件，更新 frontmatter 中 `resolved=True`，写回，**桶留在原 `dynamic/` 目录，不移动**
3. 后续 `breath()` 浮现时：该桶 `decay_engine.calculate_score()` 乘以 `resolved_factor=0.05`（若同时 `digested=True` 则 ×0.02），自然降权，最终由 decay 引擎在得分 < threshold 时归档
4. `bucket_mgr.search()` 中该桶得分乘以 0.3 降权，但仍可被关键词激活

> ⚠️ **代码 Bug B-01**：当前实现中 `update(resolved=True)` 会将桶**立即移入 `archive/`**，导致桶完全消失于所有搜索路径，与上述规格不符。需移除 `bucket_manager.py` `update()` 中 resolved → `_move_bucket(archive_dir)` 的自动归档逻辑。

**返回**：`"已修改记忆桶 abc123: resolved=True → 已沉底，只在关键词触发时重新浮现"`

#### 7b 修改元数据

```python
trace(bucket_id="abc123", name="新名字", importance=8, tags="焦虑,成长")
```

**系统内部**：收集非默认值字段 → `bucket_mgr.update()` 批量更新 frontmatter

#### 7c 删除

```python
trace(bucket_id="abc123", delete=True)
```

**系统内部**：
1. `bucket_mgr.delete("abc123")` → `_find_bucket_file()` 定位文件 → `os.remove(file_path)`
2. `embedding_engine.delete_embedding("abc123")` → SQLite `DELETE WHERE bucket_id=?`
3. 返回：`"已遗忘记忆桶: abc123"`

---

### 场景 8：记忆长期未被激活，自动衰减归档（后台 decay）

**触发方式**：服务启动后，`decay_engine.start()` 创建后台 asyncio Task，每 `check_interval_hours`（默认 24h）执行一次 `run_decay_cycle()`

**系统内部发生什么**：

1. `bucket_mgr.list_all(include_archive=False)` — 获取所有活跃桶
2. 跳过 `type in ("permanent","feel")` 或 `pinned=True` 或 `protected=True` 的桶
3. **自动 resolve**：若 `importance <= 4` 且距上次激活 > 30 天且 `resolved=False` → `bucket_mgr.update(bucket_id, resolved=True)`
4. 对每桶调用 `calculate_score(metadata)`：

   **短期（days_since ≤ 3）**：
   ```
   time_weight = 1.0 + e^(-hours/36)  (t=0→×2.0, t=36h→×1.5)
   emotion_weight = base(1.0) + arousal × arousal_boost(0.8)
   combined = time_weight×0.7 + emotion_weight×0.3
   base_score = importance × activation_count^0.3 × e^(-λ×days) × combined
   ```
   **长期（days_since > 3）**：
   ```
   combined = emotion_weight×0.7 + time_weight×0.3
   ```
   **修正因子**：
   - `resolved=True` → ×0.05
   - `resolved=True && digested=True` → ×0.02
   - `arousal > 0.7 && resolved=False` → ×1.5（高唤醒紧迫加成）
   - `pinned/protected/permanent` → 返回 999.0（永不衰减）
   - `type=="feel"` → 返回 50.0（固定）

5. `score < threshold`（默认 0.3）→ `bucket_mgr.archive(bucket_id)` → `_move_bucket()` 将文件从 `dynamic/` 移动到 `archive/` 目录，更新 frontmatter `type="archived"`

**返回 stats**：`{"checked": N, "archived": N, "auto_resolved": N, "lowest_score": X}`

---

### 场景 9：用户使用 dream 工具进行记忆沉淀

**触发**：Claude 在对话启动时，`breath()` 之后调用 `dream()`

**OB 工具调用**：`dream()`（无参数）

**系统内部发生什么**：

1. `bucket_mgr.list_all()` → 过滤非 `permanent/feel/pinned/protected` 桶
2. 按 `created` 降序取前 10 条（最近新增的记忆）
3. 对每条拼接：名称、resolved 状态、domain、V/A、创建时间、正文前 500 字符
4. **连接提示**（embedding 已开启 && 桶数 >= 2）：
   - 取每个最近桶的 embedding（`embedding_engine.get_embedding(bucket_id)`）
   - 两两计算 `_cosine_similarity()`，找相似度最高的对
   - 若 `best_sim > 0.5` → 输出提示：`"[名A] 和 [名B] 似乎有关联 (相似度:X.XX)"`
5. **feel 结晶提示**（embedding 已开启 && feel 数 >= 3）：
   - 对所有 feel 桶两两计算相似度
   - 若某 feel 与 >= 2 个其他 feel 相似度 > 0.7 → 提示升级为 pinned 桶
6. 返回标准 header 说明（引导 Claude 自省）+ 记忆列表 + 连接提示 + 结晶提示

**Claude 后续行为**（根据 CLAUDE_PROMPT 引导）：
- `trace(bucket_id, resolved=1)` 放下可以放下的
- `hold(content="...", feel=True, source_bucket="xxx", valence=0.6)` 写感受
- 无沉淀则不操作

---

### 场景 10：用户使用 feel 工具记录 Claude 的感受

**触发**：Claude 在 dream 后决定记录某段记忆带来的感受

**OB 工具调用**：
```python
hold(content="她问起了警校的事，我感觉她在用问题保护自己，问是为了不去碰那个真实的恐惧。", feel=True, source_bucket="abc123", valence=0.45, arousal=0.4)
```

**系统内部发生什么**：

1. `feel=True` → 进入 feel 专用路径，跳过自动打标和合并检测
2. `feel_valence = valence`（Claude 自身视角的情绪，非事件情绪）
3. `bucket_mgr.create(content, tags=[], importance=5, domain=[], valence=feel_valence, arousal=feel_arousal, bucket_type="feel")` → 写入 `feel/` 目录
4. `embedding_engine.generate_and_store(bucket_id, content)` — feel 桶同样有向量（供 dream 结晶检测使用）
5. 若 `source_bucket` 非空：`bucket_mgr.update(source_bucket, digested=True, model_valence=feel_valence)` → 标记源记忆已消化
   - 此后该源桶 `calculate_score()` 中 `resolved_factor = 0.02`（accelerated fade）

**衰减特性**：feel 桶 `type=="feel"` → `calculate_score()` 固定返回 50.0，永不归档
**检索特性**：不参与普通 `breath()` 浮现；只通过 `breath(domain="feel")` 读取

**返回**：`"🫧feel→<bucket_id>"`

---

### 场景 11：用户带 importance_min 参数批量拉取重要记忆

**Claude 行为**：
```python
breath(importance_min=8)
```

**系统内部发生什么**：

1. `importance_min >= 1` → 进入**批量拉取模式**，完全跳过语义搜索
2. `bucket_mgr.list_all(include_archive=False)` 全量加载
3. 过滤 `importance >= 8` 且 `type != "feel"` 的桶
4. 按 `importance` 降序排列，截断到最多 20 条
5. 对每条调用 `dehydrator.dehydrate()` 压缩，按 `max_tokens`（默认 10000）预算截断

**返回**：
```
[importance:10] [bucket_id:xxx] ...（核心原则）
---
[importance:9] [bucket_id:yyy] ...
---
[importance:8] [bucket_id:zzz] ...
```

---

### 场景 12：embedding 向量化检索场景（开启 embedding 时）

**前提**：`config.yaml` 中 `embedding.enabled: true` 且 `OMBRE_API_KEY` 已配置

**embedding 介入的两个层次**：

#### 层次 A：BucketManager.search() 内的 Layer 1.5 预筛
- 调用点：`bucket_mgr.search()` → Layer 1.5
- 函数：`embedding_engine.search_similar(query, top_k=50)` → 生成查询 embedding → SQLite 全量余弦计算 → 返回 `[(bucket_id, similarity)]` 按相似度降序
- 作用：将精排候选集从所有桶缩小到向量最近邻的 50 个，加速后续多维精排

#### 层次 B：server.py breath 的额外向量通道
- 调用点：`breath()` 检索模式中，keyword 搜索完成后
- 函数：`embedding_engine.search_similar(query, top_k=20)` → 相似度 > 0.5 的桶补充到结果集
- 标注：补充桶带 `[语义关联]` 前缀

**向量存储路径**：
- 新建桶后：`embedding_engine.generate_and_store(bucket_id, content)` → `_generate_embedding(text[:2000])` → API 调用 → `_store_embedding()` → SQLite `INSERT OR REPLACE`
- 合并更新后：同上，用 merged content 重新生成
- 删除桶时：`embedding_engine.delete_embedding(bucket_id)` → `DELETE FROM embeddings`

**SQLite 结构**：
```sql
CREATE TABLE embeddings (
    bucket_id TEXT PRIMARY KEY,
    embedding TEXT NOT NULL,   -- JSON 序列化的 float 数组
    updated_at TEXT NOT NULL
)
```

**相似度计算**：`_cosine_similarity(a, b)` = dot(a,b) / (|a| × |b|)

---

## 三、边界与降级行为

| 场景 | 异常情况 | 降级行为 |
|------|---------|---------|
| `breath()` 浮现 | 桶目录为空 | 返回 `"权重池平静，没有需要处理的记忆。"` |
| `breath()` 浮现 | `list_all()` 异常 | 返回 `"记忆系统暂时无法访问。"` |
| `breath()` 检索 | `bucket_mgr.search()` 异常 | 返回 `"检索过程出错，请稍后重试。"` |
| `breath()` 检索 | embedding 不可用 / API 失败 | `logger.warning()` 记录，跳过向量通道，仅用 keyword 检索 |
| `breath()` 检索 | 结果 < 3 条 | 40% 概率从低权重旧桶随机浮现 1~3 条，标注 `[surface_type: random]` |
| `hold()` 自动打标 | `dehydrator.analyze()` 失败 | 降级到默认值：`domain=["未分类"], valence=0.5, arousal=0.3, tags=[], name=""` |
| `hold()` 合并检测 | `bucket_mgr.search()` 失败 | `logger.warning()`，直接走新建路径 |
| `hold()` 合并 | `dehydrator.merge()` 失败 | `logger.warning()`，跳过合并，直接新建 |
| `hold()` embedding | API 失败 | `try/except` 吞掉，embedding 缺失但不影响存储 |
| `grow()` 日记拆分 | `dehydrator.digest()` 失败 | 返回 `"日记整理失败: {e}"` |
| `grow()` 单条处理失败 | 单个 item 异常 | `logger.warning()` + 标注 `⚠️条目名`，其他条目正常继续 |
| `grow()` 内容 < 30 字 | — | 快速路径：`analyze()` + `_merge_or_create()`，跳过 `digest()`（节省 token） |
| `trace()` | `bucket_mgr.get()` 返回 None | 返回 `"未找到记忆桶: {bucket_id}"` |
| `trace()` | 未传任何可修改字段 | 返回 `"没有任何字段需要修改。"` |
| `pulse()` | `get_stats()` 失败 | 返回 `"获取系统状态失败: {e}"` |
| `dream()` | embedding 未开启 | 跳过连接提示和结晶提示，仅返回记忆列表 |
| `dream()` | 桶列表为空 | 返回 `"没有需要消化的新记忆。"` |
| `decay_cycle` | `list_all()` 失败 | 返回 `{"checked":0, "archived":0, ..., "error": str(e)}`，不终止后台循环 |
| `decay_cycle` | 单桶 `calculate_score()` 失败 | `logger.warning()`，跳过该桶继续 |
| 所有 feel 操作 | `source_bucket` 不存在 | `logger.warning()` 记录，feel 桶本身仍成功创建 |
| `dehydrator.dehydrate()` / `analyze()` / `merge()` / `digest()` | API 不可用（`api_available=False`）| **直接向 MCP 调用端明确报错（`RuntimeError`）**，无本地降级。本地关键词提取质量不足以替代语义打标与合并，静默降级比报错更危险（可能产生错误分类记忆）。 |
| `embedding_engine.search_similar()` | `enabled=False` | 直接返回 `[]`，调用方 fallback 到 keyword 搜索 |

---

## 四、数据流图

### 4.1 一条记忆的完整生命周期

```
用户输入内容
     │
     ▼
Claude 决策: hold / grow / 自动
     │
     ├─[grow 长内容]──→ dehydrator.digest(content)
     │                    DIGEST_PROMPT → LLM API
     │                    返回 [{name,content,domain,...}]
     │                    ↓ 每条独立处理 ↓
     │
     └─[hold 单条]──→ dehydrator.analyze(content)
                          ANALYZE_PROMPT → LLM API
                          返回 {domain, valence, arousal, tags, suggested_name}
                          │
                          ▼
                    _merge_or_create()
                          │
                    bucket_mgr.search(content, limit=1, domain_filter)
                          │
                    ┌─────┴─────────────────────────┐
                    │ score > merge_threshold (75)?  │
                    │                                │
                   YES                               NO
                    │                                │
                    ▼                                ▼
           dehydrator.merge(              bucket_mgr.create(
             old_content, new)              content, tags,
             MERGE_PROMPT → LLM             importance, domain,
                    │                       valence, arousal,
                    ▼                       bucket_type="dynamic"
           bucket_mgr.update(...)         )
                    │                        │
                    └──────────┬─────────────┘
                               │
                               ▼
                    embedding_engine.generate_and_store(
                      bucket_id, content)
                      → _generate_embedding(text[:2000])
                      → API 调用 (gemini-embedding-001)
                      → _store_embedding() → SQLite
                               │
                               ▼
                    文件写入: {buckets_dir}/dynamic/{domain}/{name}_{id}.md
                    YAML frontmatter:
                      id, name, tags, domain, valence, arousal,
                      importance, type="dynamic", created, last_active,
                      activation_count=0   # B-04: starts at 0; touch() bumps to 1+
                               │
                               ▼
          ┌─────── 记忆桶存活期 ──────────────────────────────────────┐
          │                                                           │
          │  每次被 breath(query) 检索命中:                           │
          │    bucket_mgr.touch(bucket_id)                           │
          │      → last_active = now_iso()                           │
          │      → activation_count += 1                            │
          │      → _time_ripple(source_id, now, hours=48)           │
          │        对 48h 内邻近桶 activation_count += 0.3           │
          │                                                           │
          │  被 dream() 消化:                                        │
          │    hold(feel=True, source_bucket=id) →                  │
          │    bucket_mgr.update(id, digested=True)                 │
          │                                                           │
          │  被 trace(resolved=1) 标记:                              │
          │    resolved=True → decay score ×0.05 (或 ×0.02)        │
          │                                                           │
          └───────────────────────────────────────────────────────────┘
                               │
                               ▼
          decay_engine 后台循环 (每 check_interval_hours=24h)
            run_decay_cycle()
              → 列出所有动态桶
              → calculate_score(metadata)
                  importance × activation_count^0.3
                  × e^(-λ×days)
                  × combined_weight
                  × resolved_factor
                  × urgency_boost
              → score < threshold (0.3)?
                               │
                         ┌─────┴──────┐
                         │            │
                        YES           NO
                         │            │
                         ▼            ▼
              bucket_mgr.archive(id)  继续存活
                → _move_bucket()
                → 文件移动到 archive/
                → frontmatter type="archived"
                         │
                         ▼
             记忆桶归档（不再参与浮现/搜索）
             但文件仍存在，可通过 pulse(include_archive=True) 查看
```

### 4.2 feel 桶的特殊路径

```
hold(feel=True, source_bucket="xxx", valence=0.45)
         │
         ▼
  bucket_mgr.create(bucket_type="feel")
  写入 feel/ 目录
         │
         ├─→ embedding_engine.generate_and_store()（供 dream 结晶检测）
         │
         └─→ bucket_mgr.update(source_bucket, digested=True, model_valence=0.45)
                    源桶 resolved_factor → 0.02
                    加速衰减直到归档

feel 桶自身:
  - calculate_score() 返回固定 50.0
  - 不参与普通 breath 浮现
  - 不参与 dreaming 候选
  - 只通过 breath(domain="feel") 读取
  - 永不归档
```

---

## 五、代码与规格差异汇总（审查版）

> 本节由完整源码审查生成（2026-04-21），记录原待实现项最终状态、新发现 Bug 及参数决策。

---

### 5.1 原待实现项最终状态

| 编号 | 原描述 | 状态 | 结论 |
|------|--------|------|------|
| ⚠️-1 | `dehydrate()` 无本地降级 fallback | **已确认为设计决策** | API 不可用时直接向 MCP 调用端报错（RuntimeError），不降级，见三、降级行为表 |
| ⚠️-2 | `run_decay_cycle()` auto_resolved 实现存疑 | ✅ 已确认实现 | `decay_engine.py` 完整实现 imp≤4 + >30天 + 未解决 → `bucket_mgr.update(resolved=True)` |
| ⚠️-3 | `list_all()` 是否遍历 `feel/` 子目录 | ✅ 已确认实现 | `list_all()` dirs 明确包含 `self.feel_dir`，递归遍历 |
| ⚠️-4 | `_time_ripple()` 浮点增量被 `int()` 截断 | ❌ 已确认 Bug | 见 B-03，决策见下 |
| ⚠️-5 | Dashboard `/api/*` 路由认证覆盖 | ✅ 已确认覆盖 | 所有 `/api/buckets`、`/api/search`、`/api/network`、`/api/bucket/{id}`、`/api/breath-debug` 均调用 `_require_auth(request)` |

---

### 5.2 新发现 Bug 及修复决策

| 编号 | 场景 | 严重度 | 问题描述 | 决策 & 修复方案 |
|------|------|--------|----------|----------------|
| **B-01** | 场景7a | 高 | `bucket_mgr.update(resolved=True)` 当前会将桶立即移入 `archive/`（type="archived"），规格预期"降权留存、关键词可激活"。resolved 桶实质上立即从所有搜索路径消失。 | **修复**：移除 `bucket_manager.py` `update()` 中 `resolved → _move_bucket(archive_dir)` 的自动归档逻辑，仅更新 frontmatter `resolved=True`，由 decay 引擎自然衰减至 archive。 |
| **B-03** | 全局 | 高 | `_time_ripple()` 对 `activation_count` 做浮点增量（+0.3），但 `calculate_score()` 中 `max(1, int(...))` 截断小数，增量丢失，时间涟漪对衰减分无实际效果。 | **修复**：`decay_engine.py` `calculate_score()` 中改为 `activation_count = max(1.0, float(metadata.get("activation_count", 1)))` |
| **B-04** | 场景1 | 中 | `bucket_manager.create()` 初始化 `activation_count=1`，冷启动检测条件 `activation_count==0` 对所有正常创建的桶永不满足，高重要度新桶不被优先浮现。 | **决策：初始化改为 `activation_count=0`**。语义上"创建"≠"被召回"，`touch()` 首次命中后变为 1，冷启动检测自然生效。规格已更新（见场景1步骤6 & 场景3 create 详情）。 |
| **B-05** | 场景5 | 中 | `bucket_manager.py` `_calc_time_score()` 实现 `e^(-0.1×days)`，规格为 `e^(-0.02×days)`，衰减速度快 5 倍，30天后时间分 ≈ 0.05（规格预期 ≈ 0.55），旧记忆时间维度近乎失效。 | **决策：保留规格值 `0.02`**。记忆系统中旧记忆应通过关键词仍可被唤醒，时间维度是辅助信号不是淘汰信号。修复：`_calc_time_score()` 改为 `return math.exp(-0.02 * days)` |
| **B-06** | 场景5 | 中 | `bucket_manager.py` `w_time` 默认值为 `2.5`，规格为 `1.5`，叠加 B-05 会导致时间维度严重偏重近期记忆。 | **决策：保留规格值 `1.5`**。修复：`w_time = scoring.get("time_proximity", 1.5)` |
| **B-07** | 场景5 | 中 | `bucket_manager.py` `content_weight` 默认值为 `3.0`，规格为 `1.0`（body×1）。正文权重过高导致合并检测（`search(content, limit=1)`）误判——内容相似但主题不同的桶被错误合并。 | **决策：保留规格值 `1.0`**。正文是辅助信号，主要靠 name/tags/domain 识别同话题桶。修复：`content_weight = scoring.get("content_weight", 1.0)` |
| **B-08** | 场景8 | 低 | `run_decay_cycle()` 内 auto_resolve 后继续使用旧 `meta` 变量计算 score，`resolved_factor=0.05` 需等下一 cycle 才生效。 | **修复**：auto_resolve 成功后执行 `meta["resolved"] = True` 刷新本地 meta 变量。 |
| **B-09** | 场景3 | 低 | `hold()` 非 feel 路径中，用户显式传入的 `valence`/`arousal` 被 `analyze()` 返回值完全覆盖。 | **修复**：若用户显式传入（`0 <= valence <= 1`），优先使用用户值，`analyze()` 结果作为 fallback。 |
| **B-10** | 场景10 | 低 | feel 桶以 `domain=[]` 创建，但 `bucket_manager.create()` 中 `domain or ["未分类"]` 兜底写入 `["未分类"]`，数据不干净。 | **修复**：`create()` 中对 `bucket_type=="feel"` 单独处理，允许空 domain 直接写入。 |

---

### 5.3 已确认正常实现

- `breath()` 浮现模式不调用 `touch()`，不重置衰减计时器
- `feel` 桶 `calculate_score()` 返回固定 50.0，永不归档
- `breath(domain="feel")` 独立通道，按 `created` 降序，不压缩展示原文
- `decay_engine.calculate_score()` 短期（≤3天）/ 长期（>3天）权重分离公式
- `urgency_boost`：`arousal > 0.7 && !resolved → ×1.5`
- `dream()` 连接提示（best_sim > 0.5）+ 结晶提示（feel 相似度 > 0.7 × ≥2 个）
- 所有 `/api/*` Dashboard 路由均受 `_require_auth` 保护
- `trace(delete=True)` 同步调用 `embedding_engine.delete_embedding()`
- `grow()` 单条失败 `try/except` 隔离，标注 `⚠️条目名`，其他条继续

---

*本文档基于代码直接推导，每个步骤均可对照源文件函数名和行为验证。如代码更新，请同步修订此文档。*
