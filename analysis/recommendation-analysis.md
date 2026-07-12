# Wanso 菜品/餐厅推荐工作流分析与优化建议

> 分析对象:`server.py` 中的 `POST /api/chat` 推荐 pipeline(`server.py:2649–4926`,共约 2277 行)
> 分析日期:2026-06-14
> 配套阅读:`F:\Wanso\CLAUDE.md`、`wanso-backend/CLAUDE.md`

---

## 一、推荐工作流总览

一次 `/api/chat` 请求从客户端进入,到返回推荐卡片,完整经过 25+ 个阶段。下图是简化版的关键路径:

```
POST /api/chat
  │
  ├─[1] IP 限流(内存,_rate_limits,10/min/worker)            server.py:834
  ├─[2] GPS 兜底:缺失 → 用城市中心坐标                         server.py:2670
  ├─[3] 地点 geocode:消息含"老城区"等 → Google Maps 覆盖中心  server.py:2683
  ├─[4] 响应缓存查询(30 分钟,内存,per-worker)                server.py:2692
  ├─[5] 启发式语言检测(中文/越南语字符计数)                   server.py:2699
  ├─[6] 从 history 反推对话状态(shown_restaurants/last_action) server.py:2722
  │
  ├─[7] ★ AI 意图识别(QWEN_FAST_MODEL,temp 0.1,300 tokens,30s) server.py:2920
  │       输出 intent / food_keywords / required_elements /
  │             max_price / sort_by / is_compare /
  │             target_restaurant / user_vibe / emotional_need …
  │
  ├─[8] 比较模式三重保险(AI flag + intent==compare + 关键词)   server.py:2947
  ├─[9] ★ 硬编码 strict-food 快速路径(phở/bánh mì/bún chả)
  │       → 命中则直接 SQL 候选,跳过 [10]–[19]                  server.py:2961, 2480
  ├─[10] 餐段注入(早/午/晚/夜宵 → 注入 food_kws + sort=distance) server.py:2979
  │
  ├─[11] 路由分支:
  │     • followup   → 加载菜单 + Qwen 对话,流式返回            server.py:3026
  │     • menu_query → 3 级餐厅查找 → 拼菜单文本 + 翻译          server.py:3197
  │     • greeting/chitchat → 返回 canned 文案                  server.py:3320
  │     • search/compare → 进入 [12]
  │
  ├─[12] ★ 候选获取(PostgreSQL):
  │     a. 按 keyword 扫餐厅名+菜系(LIKE,可选 unaccent LIKE)     server.py:3507
  │     b. 按 keyword 扫菜品(menu_items JOIN,is_open=1)        server.py:3542
  │     c. R67 语义扩展:形容词(cay/sweet/healthy) → 已知菜系    server.py:3455
  │     d. compare 模式:Grab/Shopee 轮流取,平衡 top-10         server.py:4260
  │     e. 候选上限:60(alt 模式 100)                           server.py:3538
  │
  ├─[13] 过滤管道(顺序敏感):
  │     rating>0 → 菜单存在性(批量 1 query) → 菜单相关性(N+1)
  │     → 预算过滤(N+1) → 0 结果时 AI 翻译重试(1 extra AI call)
  │     → 反馈分(批量 1 query) → 终极品类门(N+1) → 距离过滤     server.py:3627–3939
  │
  ├─[14] 偏好软重排(preferences 字符串硬解析)                  server.py:3977
  ├─[15] MANDATORY 硬过滤(iOS chip)+ 优先级回退              server.py:4079
  ├─[16] PRICE-SORT FIX:重查最便宜匹配菜再排序                server.py:4153
  ├─[17] menu_sample 抓取(top-15 候选,每个 1+ queries)        server.py:4184
  ├─[18] compare 模式:平台分组 + 跨平台 PRICE VERDICT          server.py:4313
  │
  ├─[19] ★ 构造 system_prompt + user_prompt(currency/dist/
  │       compound/vibe/alt/fallback contexts 全部拼进去)      server.py:4405
  │
  ├─[20] ★ AI 推荐(QWEN_MODEL,temp 0.85,top_p 0.92,
  │       1200–1500 tokens,120s timeout,流式)                 server.py:4544
  │
  ├─[21] parse_ai_response(JSON 解析,index 边界校验)           server.py:4560
  │
  ├─[22] ★ 每个 pick 后处理(每张卡片 3+ queries):
  │       full row + avg/min price + sample_dishes
  │       → 相对排名 → recommend_tags(nearby/value/popular/fresh)
  │       → flex_tags(多级兜底)
  │       → normalize_cuisine_labels(可能再调 AI)              server.py:4635
  │
  ├─[23] R98 picks 内部标签重打(避免雷同)                      server.py:4748
  ├─[24] R82 给 picks[0] 加 smart + 竞争优势前缀                server.py:4830
  └─[25] text 拼接 [id:N] 标记(供下轮上下文)+ cache put + 返回  server.py:4886
```

一次成功 chat 至少 **2 次 Qwen 调用**(意图 + 推荐),典型 **3–4 次**(+ 翻译重试 / cuisine 标签),DB 查询 **60–200+ 次**(下面 §2.1 详述)。

---

## 二、性能问题(延迟 / DB 压力)

### 2.1 ⚠️ **N+1 查询是最大瓶颈**(P0)

#### N+1 查询

**正式名称**:**N+1 查询问题**(_N+1 Query Problem_),也叫 **N+1 selects 问题**。也叫 **"循环查询"** 或 **"循环内查数据库"**。

**一句话定义**:查一个列表用 1 次 SQL,然后**对列表里每一项再各发 1 次 SQL** 去取关联数据。总共 `1 + N` 次往返。

**通俗比喻(用项目本身的例子)**:用户搜 "phở / bún / cơm",系统找出 50 家候选餐厅,现在要给每家挑一道代表菜——

- ❌ **错误做法(现在的代码)**:跑到第一家,翻它菜单问"有 phở 吗?"→ 没有 → 再问"有 bún 吗?"→ 没有 → 再问"有 cơm 吗?"……每家餐厅翻 3 次菜单。然后跑第二家、第三家……50 家 × 3 关键词 = **150 次单独翻菜单**。
- ✅ **正确做法**:写一条 SQL 一次查"这 50 家里所有名字含 phở / bún / cơm 的菜,每家给我一道最便宜的"。**1 次查完**。
- 数据量一模一样,时间差几十倍。

**为什么这么贵**:数据库单次查询的成本主要是**网络往返时间**(RTT,~1–3ms)+ SQL 解析/锁/执行(~5–10ms)。即使每次只取 1 行,固定开销也跑不掉。所以 150 次查询 ≈ 1 秒以上,1 次查询 ≈ 10ms。

**自查规则**:Python 层 `for` 循环里出现 `db.execute(...)` 的地方,几乎一定是 N+1。

---

#### `get_best_dish` 是**两层** N+1 套娃

你问的 `get_best_dish`,答案是"是,而且更糟":

**内层(函数本身,server.py:545)** —— 对每个关键词查 1 次:

```python
def get_best_dish(db, restaurant_id, search_kws):
    for kw in search_kws:                       # ← 内层:每个关键词
        matched = db.execute(                   # ← 每关键词 1 次 SQL
            "SELECT name, price FROM menu_items WHERE restaurant_id = ? AND name LIKE ?", ...)
        if dish: break                          # 命中就停
    if not dish:
        mains = db.execute(...)                 # fallback 1,又 1 次
    if not dish:
        cheapest = db.execute(...)              # fallback 2,又 1 次
```

一次调用最多:`len(search_kws)` + 2 次 SQL。

**外层(调用处,server.py:3528)** —— 又对每家候选调用一次:

```python
for kw in all_kws[:15]:              # 外层 keyword
    for plat in search_platforms:    # × 平台
        rows = db.execute(...)       # 找候选餐厅
        for r in rows:               # × 每家候选(~20 家)
            dish = get_best_dish(db, r["id"], all_kws)   # ← 外层 N+1
```

理论最坏:`15 关键词 × 2 平台 × 20 候选 × 17 内层查询 ≈ 1 万次`。实际有 `seen` 去重 + 早 break,落到 60–180 次(见下表第一行)。

**修复 = 一次性查所有(候选 × 关键词)组合**:

```sql
SELECT DISTINCT ON (restaurant_id) restaurant_id, name, price
FROM menu_items
WHERE restaurant_id IN (?, ?, ?, ...)                       -- 所有候选餐厅
  AND (name LIKE '%pho%' OR name LIKE '%bun%' OR ...)       -- 所有关键词一起 OR
ORDER BY restaurant_id, price ASC
```

**1 次 SQL 代替几千次**。

---

#### 当前项目里的 N+1 分布

候选获取/打分阶段存在严重的 N+1 模式,典型一次 chat 会产生 **60–200+ DB 查询**:

| 位置                                                          | 问题                                                                                    | 次数估算                                                        |
| ------------------------------------------------------------- | --------------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| `server.py:3528` `get_best_dish(db, r["id"], all_kws)`        | **嵌在 keyword 循环里**,每个 keyword×平台 都跑,内部又是 3 级 fallback                   | 15 keywords × 2 platforms × ~20 命中 × 1–3 queries = **60–180** |
| `server.py:3650` `calculate_menu_relevance(db, c["id"], ...)` | 每个候选 1 次 `SELECT * FROM menu_items WHERE restaurant_id=?`,然后 Python 内存循环匹配 | 15–60 候选 = **15–60**                                          |
| `server.py:3718` 预算过滤                                     | 每个 keyword 每个候选 1 次 `COUNT(*)` 查询                                              | 5 kws × 15 候选 = **75**                                        |
| `server.py:3875` 终极品类门                                   | 每个候选 1 次 `SELECT name FROM menu_items WHERE restaurant_id=?`(全表!)                | 15 候选 = **15**                                                |
| `server.py:4637–4657` 每个 pick 后处理                        | full row + avg/min price + sample dishes = 3 queries                                    | 5 picks × 3 = **15**                                            |

**估算总查询数**:典型路径 60–200 次,极差路径 300+。

**修复思路(按收益排序)**:

1. **`get_best_dish` 移出 keyword 循环**:候选收集完后,一次性用 `WHERE restaurant_id IN (...) AND (name LIKE ? OR ...)` 拿到所有候选的最佳匹配菜。
2. **`calculate_menu_relevance` 批量化**:用 `SELECT restaurant_id, name FROM menu_items WHERE restaurant_id IN (...) AND price >= ?` 一次拿所有候选菜单,Python 内 group_by 后打分。
3. **预算过滤批量化**:`SELECT restaurant_id, COUNT(*) FROM menu_items WHERE restaurant_id IN (...) AND price BETWEEN ? AND ? AND (name LIKE ? OR ...) GROUP BY restaurant_id`。
4. **终极品类门可以和 `calculate_menu_relevance` 合并**:两者都在看 menu_items 表,只差过滤条件。

> **预期收益**:DB 查询数从 200+ 降到 10–15,P95 延迟可降 1–3 秒(取决于网络 RTT × 17 workers 抢锁)。

### 2.2 ⚠️ 严格的 fast-path 必须等 AI 意图调用完成才触发(P0)

#### 先讲清楚:fast-path 是什么

**fast-path** 是通用编程术语:**为常见场景写的"捷径"代码,跳过通用(但慢)的处理流程**。

在本项目里,fast-path 具体指 **`_strict_food_terms` 这段硬编码检测**(server.py:856)——只认得 3 个越南菜(phở / bánh mì / bún chả),但这 3 个菜占了用户搜索的相当大比例。

> 注意:fast-path ≠ 关键词匹配。**关键词匹配只是检测方式,fast-path 的核心价值是跳过 AI 调用**。如果只是关键词匹配完还是要调 AI,那不叫 fast-path。

#### 一次 chat 有两次 AI 调用

```
请求 → [AI #1: 意图识别]   ~1–3 秒   ← Qwen FAST model(server.py:2920)
       ↓
       [AI #2: 推荐生成]   ~3–8 秒   ← Qwen 主模型(server.py:4544)
       ↓
       返回结果
```

#### 现在的 fast-path 只跳过了 AI #2,没跳过 AI #1

看代码顺序:

```python
# 1. 先调 AI #1(永远跑,~1–3 秒)
intent_resp = await asyncio.to_thread(
    lambda: qwen_client.chat.completions.create(model=QWEN_FAST_MODEL, ...))
intent = json.loads(...)

# 2. AI #1 跑完之后,才检查 fast-path
if intent.get("intent") in ("search", "compare") and not intent.get("target_restaurant"):
    _strict_food_resp = _strict_food_chat_response(...)   # ← fast-path(server.py:2961)
    if _strict_food_resp is not None:
        return _strict_food_resp   # 直接 SQL 出结果,跳过 AI #2
```

| AI 调用 | fast-path 命中时 |
|---|---|
| **#1 意图识别**(server.py:2920) | ❌ **照常调用,1–3 秒照花** |
| **#2 推荐生成**(server.py:4544) | ✅ 跳过(直接 SQL 出结果) |

#### 为什么现在写法必须等 AI #1 才查 fast-path

代码里 fast-path 检查的入参有个判断:`not intent.get("target_restaurant")`。意思是"如果用户在追问之前看过的某家店(比如'第二家怎么样'),别走 fast-path,要走 followup"。

问题在于:`target_restaurant` 这个字段是 AI #1 返回的——为了知道要不要走 fast-path,**必须先跑 AI #1**。鸡生蛋。

#### 三种情形的延迟对比

用户搜 "phở":

| 情形 | AI #1 | AI #2 | SQL | 总耗时 |
|---|---|---|---|---|
| 没有 fast-path | 1–3s | 3–8s | ~100ms | **4–11 秒** |
| **现在(部分 fast-path)** | 1–3s | 跳过 | ~100ms | **1–3 秒** |
| **§2.2 优化后(完全 fast-path)** | 跳过 | 跳过 | ~100ms | **~100 毫秒** |

省下的 1–3 秒就是 AI #1 那一调用。

#### 修复:用本地启发式判断"是不是 followup",把 fast-path 真正前置

```python
# 在调 AI #1 之前,先看一眼消息本身
def looks_like_followup(message):
    return any(w in message.lower() for w in [
        "第一家", "第二家", "第三家", "first one", "second one",
        "那家", "这家", "第 1", "menu của", "菜单",
        "另一个", "另外", "alternative",
    ])

# fast-path 真前置
if not looks_like_followup(message):
    strict_terms = _strict_food_terms(message)
    if strict_terms:
        # 命中 → 完全跳过 AI,直接 SQL
        return _strict_food_chat_response(message, city, ...)

# 没命中 fast-path,才走 AI #1
intent_resp = await ...
```

启发式判断不完美,但 fast-path 命中的场景(用户明确说想吃 phở / bánh mì / bún chả)通常也不会同时是 followup。**错判的兜底是 fast-path 不命中,继续走 AI,不会出错——只是没省到而已**。

#### 一句话汇报

> 现在的 fast-path 只跳过了一次 AI 调用(节省 3–8 秒),但**第一次 AI 调用(意图识别,1–3 秒)无论如何都会跑**。改造成"先看消息再决定要不要调 AI",phở / bánh mì / bún chả 这类高频查询能再省 1–3 秒。

### 2.3 city-center 兜底时不加 bbox(P1)

`server.py:3578` 当 GPS 缺失且 `sort_by != "distance"` 时,SQL 没有 lat/lng bbox,会扫整个城市。河内/胡志明有几万家餐厅,这种全表扫描在 PG 上也很慢。

**修复**:即使按 rating 排序,也应该用 city-center 的默认 bbox(±0.05°)约束。

### 2.4 多次顺序 AI 调用串行执行(P1)

典型路径:意图(~1–3s)→ 推荐(~3–8s),如果走翻译重试再加 1 次,如果 pick 调 `normalize_cuisine_labels` 还要 5 次。这些是**严格串行**的。

**优化方向**:

- `normalize_cuisine_labels` 已经走的是字典(`VI_CUISINE_MAP`),只在字典未命中时才调 AI——这部分已经做得不错,但需要确认实际命中率(如果命中率 >80% 就基本无影响)。
- 翻译重试可以提前:在 AI 意图阶段就同时返回 `vi_keywords`(让意图模型一次给两套),省一次往返。

### 2.5 内存缓存是 per-worker,17 workers 各一份(P2)

`_chat_cache_*`、`_rate_limits`、`_restaurants_with_menu_cache`、`_weather_cache`(scenario_engine)、`_cross_menu_cache`(server.py:1148)都是模块级 dict,17 个 gunicorn workers 各自一份。

- 响应缓存命中率 = `1 / 17` 上界(同一条消息被 hash 到不同 worker 就不命中)。
- 天气缓存理论 30 分钟一查,实际可能 30 秒一查(每个 worker 独立 TTL)。

**优化方向**:

- 加 Redis(后端同机部署,本地 socket,延迟 <1ms)。
- 或:启用 gunicorn `--preload` + 共享内存(但 Python GIL 限制大,不推荐)。
- 短期方案:在 nginx/前置反代层做 `hash(message) % 17 → 路由到固定 worker`,把缓存命中率拉回 1/1。但牺牲负载均衡。

---

## 三、成本问题(Qwen token / 调用次数)

### 3.1 ⚠️ 意图识别 prompt 破坏了 Qwen 的 prefix cache(P0)

`server.py:2770–2911` 的 `intent_prompt` 有 142 行,涵盖 6 个 intent + 17 个字段定义 + 4 个 CRITICAL 子规则。**问题不在长度,在于构造方式**。

**先澄清一个事实**:意图 prompt **已经是 `system` 角色**(`server.py:2913`):

```python
intent_msgs = [{"role": "system", "content": intent_prompt}]
```

所以"把 prompt 放进 system"这件事**已经做了**。真正的问题是:

#### 3.1a system 消息里被 f-string 拼进了动态变量

`server.py:2770` 的 `intent_prompt` 用的是 `f"""..."""`,里面塞了三个动态字段:

```python
intent_prompt = f"""You are Wanso's intent classifier ...
Analyze the user's message and conversation state ...

CONVERSATION STATE:
{state_text}              ← 每次请求都不一样

Current time: {time_info}              ← 每秒在变
App language setting: {lang_name}      ← 因用户而异

Return ONLY valid JSON:
...
FIELD DEFINITIONS:    ← 后面 ~3500 tokens 才是纯静态规则
...
"""
```

`state_text` / `time_info` / `lang_name` 出现在 system prompt 的**前几百个 token 位置**。

#### 3.1b DashScope 的 prompt cache 是严格前缀匹配

阿里云 DashScope 的 context cache(Qwen 的 prompt cache)按文档([help.aliyun.com/zh/model-studio/context-cache](https://help.aliyun.com/zh/model-studio/context-cache))工作方式:

- 从 token 0 开始**逐 token 比对**
- 一旦遇到第一个不同的 token,从那个位置往后**全部 miss**
- 命中粒度最小 256/1024 tokens(视模型)

所以当前实际效果:

```
[system]  You are Wanso... (~600 tokens 固定)        → ✅ cache hit
          CONVERSATION STATE: {state_text}           → ❌ 从这里开始全 miss
          Current time: 2026-06-14 16:42...          → ❌
          ... 剩下 ~3500 tokens 静态规则              → ❌ 本可缓存,但已被前段破坏
```

**结果**:~4000 tokens 的 system prompt,实际只有最前面 ~600 命中 cache,**~85% miss**。

#### 3.1c 修复:把动态变量挪到 user 消息

```python
# system: 纯静态,整段 cacheable
INTENT_SYSTEM = """You are Wanso's intent classifier for a Vietnamese food delivery
recommendation system (GrabFood + ShopeeFood).

Return ONLY valid JSON:
{
  "intent": "search|compare|...",
  ...
}

FIELD DEFINITIONS:
... (所有规则、字段、example 全部固定写死,不带任何 f-string 插值)
"""

# user: 动态内容 + 实际消息
intent_user = f"""CONVERSATION STATE:
{state_text}

Current time: {time_info}
App language setting: {lang_name}

User message: {message}
"""

intent_msgs = [
    {"role": "system", "content": INTENT_SYSTEM},   # ← 整段 cacheable
    *[{"role": h["role"], "content": h["content"][:2000]} for h in history[-6:]],
    {"role": "user", "content": intent_user},
]
```

注意 `history[-6:]` 仍然会让 cache 在 system 段之后分叉,但 system 整段固定了,**前 ~3500 tokens 始终命中**——这就是把动态内容挪走的全部价值。

#### 3.1d 预期收益

- system prompt ~3500 tokens 走 cache,DashScope 的 cache 价格通常是正常输入的 ~40%
- 每个 chat 请求省 ~2100 tokens 的全价费用
- 主要价值在 **TTFT(首 token 延迟)降 200–500ms**(cache 命中时 prefill 跳过)
- 假设日均 10k 次意图调用,按 qwen-flash 价格一年省几百到一两千 RMB

### 3.1e (次要)prompt 还可以瘦身(P1)

除了 cache 问题,这 142 行 prompt 本身也有冗余:

- 多个 example 重复表达同一规则
- `CORE PRINCIPLE` / `VERBS DETERMINE CATEGORY` / `CRITICAL` 段落有交叉

瘦身后 system 可压到 ~1500 tokens,token 成本再降一半。但这是 **3.1c 的后续优化**,先做 3.1c 修 cache,再考虑瘦身——两件事互不依赖,但顺序上 3.1c 收益更大、风险更低。

### 3.2 RANKING_PROMPT / COMPARE_PROMPT 没审计(P1)

`server.py:706`(RANKING)和 `server.py:790`(COMPARE)的 prompt 都很长,且每次都会把 `candidate_text`(15 候选 + menu_sample,~2–4KB JSON)塞进去。建议:

- 抓一份实际 prompt,用 tiktoken 数 token,看是否触碰模型上下文上限。
- COMPARE 模式可以只发 `compare_summary` 不发 `candidate_text`(代码注释里也提到了这点,server.py:4521)。

### 3.3 normalize_cuisine_labels 在 pick 循环里串行(P1)

`server.py:4725` 给每个 pick 调一次。如果 5 个 pick 都未命中字典,就是 5 次 Qwen 串行调用。可批量化(1 次调用,5 个候选一起处理)。

### 3.4 跨平台菜单对齐 `_ai_align_menus` 没复用结果(P2)

`server.py:5151` 有 7 天缓存 `_cross_menu_cache`,但只对 `/api/restaurant/{rid}` 详情页的 compare 路径有用。chat 流程里的 compare 模式走的是另一条路径(`PRICE VERDICT`,server.py:4333)用的是字符串精确匹配,几乎不可能跨平台命中(Grab 和 Shopee 的菜名很少一样)。

---

## 四、推荐质量问题

### 4.1 ⚠️ `seen` 用 `name_platform` 做 key,而非 restaurant id(P0)

`server.py:3523`、`server.py:3637`:`key = f"{r['name']}_{r['platform']}"`。

问题:

- **同品牌多分店被合并**:越南的 Phúc Long、Highlands Coffee 在同城市有几十家分店,同名同平台 → 只保留第一个,丢掉附近的分店。
- **不同餐厅同名被合并**:河内有几十家叫 "Phở 10"、"Phở Bát Đàn" 的店,key 相同就被去重了。

**修复**:用 `restaurant_id`(已经是唯一主键)做 key。如果担心 name-search 和 menu-search 同一餐厅重复加入,那也是 id 重复,id 去重更准确。

### 4.2 strict-food 检测覆盖太窄(P1)

只覆盖 phở/bánh mì/bún chả(server.py:856–910)。其他越南高频菜——bún bò Huế、cơm tấm、gỏi cuốn、bánh xèo、miến xào、chả cá——全部走慢速 AI 路径。

**修复**:扩 `_strict_food_terms`,加上 5–10 个高频越南菜。每加一个对应单测验证排除词(`exclude_norm`)和强匹配词(`strong_food_norm`)。

### 4.3 AI 返回的 food_keywords 和原始消息被混着(P1)

`server.py:3342–3353` 在 AI 返回 keywords 之后,又从 `message` 里正则抽出更多词并加进去,逻辑是"防止 AI 漏掉用户直接输入的越南语食物名"。但这个 regex `[a-zA-ZÀ-ỹ]{2,}` 不区分词性,容易把"thích"(喜欢)、"không"(不)这种非食物词也加进去。

**修复**:用更严的词表(STOP_WORDS 已经存在,server.py:494)做白名单;或者干脆信任 AI 输出。

### 4.4 `term_matches` 在 `calculate_menu_relevance` 里没用(P1)

`server.py:586` 的 `term_matches` 函数有词边界 + 声调降级,写得相当严谨。但 `calculate_menu_relevance`(server.py:604)虽然调用了它,外层循环 `for kw in food_kws: for name in menu_names` 是 **O(kws × menu_items)** 纯 Python,菜单 200 项 × 5 kws = 1000 次匹配/候选 × 15 候选 = 15000 次 regex。

**修复**:把整个匹配下推到 SQL:`SELECT restaurant_id, count(*) FILTER (WHERE name ~* ?) FROM menu_items WHERE restaurant_id IN (...) GROUP BY restaurant_id`,Python 只算分数。

### 4.5 PRICE VERDICT 几乎永远走 fallback(P1)

`server.py:4356` 要求两平台**店名和菜名都精确(去重音)匹配**才走主路径。Grab 和 Shopee 的店名很少一致(Shopee 上常带 ` - HCMC` 后缀),菜名更不一致。结果几乎都是 `_best is None`,走 `_cheapest` 兜底(server.py:4375),用不同菜的入门价比——结论虽然"对"但说服力弱。

**修复**:

- 短期:在 verdict 文案里诚实标注"comparing different dishes",别让 LLM 误以为是同菜比价。
- 长期:接 `_ai_align_menus` 的结果(server.py:5151),用 AI 对齐过的同菜对来比。

### 4.6 用户行为数据完全没用进推荐(P2)

analytics 模块(`analytics.py`)收集了 sessions / search_events / click_events / order_jumps / preference_history 等 10+ 张表。但推荐 pipeline 里**只用了 `restaurant_feedback`(likes/dislikes,server.py:3787)**,没有任何个性化。

10+ 张埋点表的成本(写 PG、占空间、维护 schema)和当前收益严重不匹配。

**优化方向**(任选):

- 把 click_events / order_jumps 聚合成"用户最近 30 天偏好菜系",chat 时拼进 system_prompt。
- 或干脆砍掉埋点(如果短期内不打算用)。

---

## 五、可维护性

### 5.1 ⚠️ 2277 行的 `chat()` 函数(P0)

`server.py:2649–4926` 单个 async 函数。代码里遍布 `# R<num>` 标记(R36、R45、R63、R64、R66–R114……),每个标记代表一次线上修复——说明这块代码修改风险极高,回归频繁。

**重构方向**(渐进,不破坏):

1. 抽出 `intent_classification()`(server.py:2920 块)。
2. 抽出 `gather_candidates(db, intent, ...)`(server.py:3431–3810 块)。
3. 抽出 `apply_filters(candidates, intent, ...)`(server.py:3627–3939 块)。
4. 抽出 `build_prompt(candidates, intent, ...)`(server.py:4405 块)。
5. 抽出 `enrich_picks(picks, candidates, ...)`(server.py:4635 块)。

每抽一层加单测。重构本身不改外部行为,但能把后续每次改动的爆炸半径缩小。

### 5.2 候选 dict 构造重复 5+ 次(P1)

12 字段的 candidate dict 在 `server.py:3530, 3566, 3591, 3615, 3774, 3832` 等处逐字复制。任何字段增删要改 5+ 处。

**修复**:抽 `_make_candidate(row, idx, user_lat, user_lng, dish=None)` 一个工厂函数。

### 5.3 Magic numbers 散落(P2)

- `MIN_DISH_PRICE = 10000`(server.py:96),但 `server.py:4197/4658` 用了硬编码 `25000`,`server.py:556` 用 `25000–150000` 区间。
- `REGION_AVG_REVIEWS = 300`、`REGION_REVIEW_P20 = 10`(server.py:97–98):胡志明和河内的评论分布不同,用同一个阈值不合理。
- `0.027`/`0.03`(server.py:3503):bbox 偏移,既不是 km 也不是 mile,直接是度,可读性差。

**修复**:全部抽成模块顶部命名常量,按城市分桶。

### 5.4 过滤管道顺序不合理(P1)

当前顺序:`gather → rating>0 → 菜单存在 → 菜单相关性 → 预算 → 翻译重试 → 反馈 → 终极品类门 → 距离`。

问题:菜单相关性和终极品类门都是 **N+1 重查询**,而预算过滤和距离过滤便宜得多,应该先跑。距离过滤尤其应该尽早跑——很多候选在 6km 外,后续所有打分都白做。

**修复**:重排为 `gather → rating>0 → 距离 → 菜单存在(批量) → 预算 → 菜单相关性(批量) → 终极品类门 → 翻译重试 → 反馈`。

### 5.5 crash_reports 表被两处创建(P2)

`server.py:5323` 和 `auth.py:265` 都 `CREATE TABLE IF NOT EXISTS crash_reports`,但 server.py 用 `extra_info`/`os_version` 列,auth.py 用 `extra`/`android_version` 列——schema 不一致。两个模块都能写,字段对不齐会让其中一边的 INSERT 默默失败或写错列。

**修复**:统一 schema,只保留一处创建。

---

## 六、缓存与状态

| 缓存                  | 位置                  | TTL    | 共享性     | 问题                           |
| --------------------- | --------------------- | ------ | ---------- | ------------------------------ |
| chat response         | server.py:5207        | 30 min | per-worker | 17 workers,实际命中率 <10%     |
| restaurants_with_menu | server.py:102         | 5 min  | per-worker | 同上                           |
| rate_limits           | server.py:833         | 1 min  | per-worker | 限流可被绕过(换 worker 就重置) |
| weather               | scenario_engine.py:12 | 30 min | per-worker | 17 倍 QPS 到 weather API       |
| cross_menu            | server.py:1148        | 7 day  | per-worker | 只在详情页用                   |

### 6.1 每个缓存具体存的是什么

#### `_chat_cache_*`(响应缓存) — server.py:5207

**存什么**:整个 chat 推荐的**返回结果**。

```python
# key
hash(message + city + lat + lng + preferences)   # 例: "a3f8c2..."

# value
{
  "text": "牛肉粉我帮你定了 - Grab 上这家评分最高...",
  "restaurants": [...5 个候选餐厅的完整 JSON...]
}
```

**为什么存**:用户连续两次问同样的问题(比如手滑重发),30 分钟内直接返回旧答案,**省 4–10 秒 AI 调用**。

**miss 的代价**:重新跑完整 pipeline。**5 个缓存里最贵的一个**。

---

#### `_restaurants_with_menu_cache`(有菜单的餐厅集合) — server.py:102

**存什么**:**所有"至少有一道价格 ≥ 10000 VND 的菜"的餐厅 ID 集合**。

```python
{"ids": {"grab-12345", "shopee-67890", ...}, "timestamp": 1718360000.0}
# 几万个 ID 的 set
```

**为什么存**:每次搜索都要快速判断"这家餐厅是不是空壳"(没有菜单的店不推荐)。如果每次都现查,要扫整个 `menu_items` 表(可能上百万行)。

**miss 的代价**:5 分钟一次的全表 DISTINCT 查询。

---

#### `_rate_limits`(限流计数) — server.py:833

**存什么**:**每个客户端 IP 最近 60 秒的请求时间戳列表**。

```python
{
  "192.168.1.1": [1718360000.0, 1718360015.0, 1718360042.0, ...],
  "10.0.0.5":    [1718360010.0, ...],
}
```

**为什么存**:防止单个 IP 在 1 分钟内调超过 10 次 `/api/chat`(防爬虫、防恶意刷 AI 调用烧钱)。

**miss 的代价**:**限流被绕过**。因为 17 个 worker 各算各的,实际允许的 QPS 是 `10 × 17 = 170/min`,而不是 10/min。

> 这是 5 个缓存里**最危险的 miss**——不是性能问题,是**安全问题**。

---

#### `_weather_cache`(天气) — scenario_engine.py:12

**存什么**:**每个城市的当前天气**(温度、是否下雨、天气状况)。

```python
{
  "hcmc":  (1718360000.0, {"temp_c": 32, "is_rain": False, "condition": "Sunny", "code": 1000}),
  "hanoi": (1718360050.0, {"temp_c": 28, "is_rain": True, "condition": "Rain", "code": 1183}),
}
```

**为什么存**:`/api/explore/scenarios` 端点根据天气决定推荐什么(下雨推热汤、高温推冷饮)。天气 API 是付费的(weatherapi.com,API key 硬编码在 scenario_engine.py:9),30 分钟 TTL 限制调用频率。

**miss 的代价**:**17 倍 API 调用费用**。理论 30 分钟 1 次,实际可能 30 秒 1 次(17 个 worker 各自的 TTL 都到期)。

---

#### `_cross_menu_cache`(跨平台菜单对齐) — server.py:1148

**存什么**:**AI 对齐过的"同一家餐厅在 Grab 和 Shopee 两边菜单的菜对"**。

```python
# key
(grab_restaurant_id, shopee_restaurant_id)

# value(假设 Phúc Long 在两边的菜单对齐结果)
[
  {"grab_name": "Trà sữa Phúc Long",   "grab_price": 45000,
   "shopee_name": "Milk Tea Phuc Long", "shopee_price": 42000},
  ...
]
```

**为什么存**:跨平台比价需要知道"这两道菜是不是同一道"。AI 对齐一次很贵(让 Qwen 看 100 道菜的两份菜单),所以缓存 7 天。

**miss 的代价**:每次用户开"同一家在两个平台"的对比页,都重跑 AI 对齐。**这个只在 `/api/restaurant/{rid}` 详情页用,不在主 chat 流程**。

---

### 6.2 一句话总结(可放汇报)

| 缓存 | 缓的是什么 | miss 的代价 |
|---|---|---|
| chat response | 完整推荐结果(文字 + 5 张餐厅卡) | 4–10 秒 AI 重跑 |
| restaurants_with_menu | "有菜单的餐厅 ID 集合"(几万个 ID) | 5 分钟一次全表扫描 |
| rate_limits | 每 IP 最近 60 秒的请求时间戳 | **限流被绕过 17 倍** ⚠️ |
| weather | 每城市当前天气 | 17 倍付费 API 调用 |
| cross_menu | AI 对齐过的跨平台菜单对 | 详情页每次重跑 AI 对齐 |

---

### 6.3 根本性问题与改造方向

**根本性问题**:部署是 bare-metal systemd 17 workers,所有 in-process 缓存的实际命中率都被稀释 17 倍。等于花钱买了一份缓存,效果只用了 6%。

**短期改造**:Redis on localhost。所有上述缓存改成 `redis.get/set`。Redis 单实例在本地 socket 上 P99 < 1ms,远小于一次 Qwen 调用。

---

## 七、安全与边界情况

### 7.1 ⚠️ `/api/crash/list` 用硬编码口令(P0)

`server.py:5374`:`if secret != "wanso2026"`。口令已经进 git 历史,任何人 clone 仓库就能看到。即使只在内部用,也算泄露。

**修复**:迁到环境变量(已经有 `ADMIN_REFRESH_SECRET` 模式可参考),并 `del` 老 secret。

### 7.2 chat cache key 含 lat/lng(P1)

`server.py:5207` 把 `(message, city, lat, lng, preferences)` 全拼进 key。两个用户在同一城市发同一条消息但 GPS 略不同 → cache miss。GPS 兜底时用的是 city center,如果两个用户都无 GPS,反而会 cache hit 但返回基于城市中心的结果——这种结果对两边都不准。

**修复**:key 里把 lat/lng 取整到 2 位小数(~1km 精度)再 hash,牺牲少量精度换命中率。

### 7.3 翻译重试无 timeout(P1)

`server.py:3746`:`qwen_client.chat.completions.create(...)` 没传 `timeout=`,默认 120s(qwen_client 构造时设的)。如果 Qwen 慢,用户卡 2 分钟。

**修复**:显式 `timeout=15`,失败就跳过(代码已经 `except Exception: pass`,只是补 timeout)。

### 7.4 Google Maps API key 应在 .env(P2)

`server.py:2296` 附近调用 Google Maps,实际 key 应该已经在 `.env`(由 `GOOGLE_MAPS_API_KEY` 注入),需要确认没有硬编码。

---

## 八、优先级排序

| 优先级 | 项目                                                              | 位置      | 预期收益                                        | 工作量                   |
| ------ | ----------------------------------------------------------------- | --------- | ----------------------------------------------- | ------------------------ |
| **P0** | N+1 查询批量化(get_best_dish、calculate_menu_relevance、预算过滤) | §2.1      | DB 查询数 -90%,P95 延迟 -1–3s                   | M(2–3 天)                |
| **P0** | strict-food 快速路径前置到 AI 调用之前                            | §2.2      | phở/bánh mì 查询 -1–3s                          | S(半天)                  |
| **P0** | 候选去重 key 改用 restaurant_id                                   | §4.1      | 修复多分店丢失 bug                              | S(2 小时)                |
| **P0** | `/api/crash/list` 硬编码口令迁 env                                | §7.1      | 安全                                            | S(1 小时)                |
| **P0** | 意图 prompt 把动态变量挪到 user 消息,修 prefix cache 失效         | §3.1a–d   | system 段 cache 命中 15% → 100%,TTFT -200–500ms | S(2 小时)                |
| **P1** | city-center 兜底加 bbox                                           | §2.3      | 全表扫描 → bbox 扫描                            | S(2 小时)                |
| **P1** | 意图 prompt 文案瘦身(4000 → ~1500 tokens)                         | §3.1e     | system cache 命中后再省 50% token               | M(1 天,需对比效果)       |
| **P1** | 过滤管道顺序重排(便宜 filter 先跑)                                | §5.4      | 候选打分次数 -50%                               | S(半天)                  |
| **P1** | strict-food 扩到 8–10 个高频菜                                    | §4.2      | 更多查询走快速路径                              | S(1 天,需对每菜写排除词) |
| **P1** | normalize_cuisine_labels 批量化                                   | §3.3      | 省 4 次 AI 串行                                 | S(半天)                  |
| **P1** | 翻译重试补 timeout                                                | §7.3      | 防 120s 卡死                                    | XS(15 分钟)              |
| **P1** | chat cache key 把坐标取整                                         | §7.2      | 命中率提升                                      | XS(15 分钟)              |
| **P2** | 引入 Redis 替代 per-worker 内存缓存                               | §2.5 / §6 | 缓存命中率 +10x                                 | L(2–3 天,需 ops)         |
| **P2** | 把 click/order 埋点接进推荐                                       | §4.6      | 真正个性化                                      | L(1 周)                  |
| **P2** | `chat()` 拆 5 个子函数                                            | §5.1      | 后续改动风险 -50%                               | L(1 周,需回归测试)       |
| **P2** | PRICE VERDICT 接 AI 菜单对齐                                      | §4.5      | compare 结论更可信                              | M(2 天)                  |
| **P2** | crash_reports schema 统一                                         | §5.5      | 防 silent 失败                                  | S(2 小时)                |
| **P2** | 候选 dict 工厂函数                                                | §5.2      | 维护性                                          | S(2 小时)                |

P0 全做完:typical chat 延迟可降 30–50%,DB 负载降 60%+,修复 1 个数据正确性 bug + 1 个安全 issue。

---

## 九、给后续工作的建议

1. **先加可观测**:在 `chat()` 入口和每个过滤段埋 `print(f"TIMING stage=X ms=Y")`(部分已有,server.py:4558)。没有基线就不知道优化有没有效果。
2. **建一个 chat 请求回放集**:抓 50–100 条真实线上 chat 请求(脱敏),作为回归基准。任何 prompt/过滤改动都跑一遍,对比延迟、token、候选数。
3. **P0 改动建议拆 5 个独立 commit/PR**:每个 P0 单独可回滚。
4. **`# R<num>` 标记保留**:这些是过去踩坑的痕迹,改相关代码时先 `git log -S R67` 看上下文。
