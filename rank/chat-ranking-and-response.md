# Chat 排序与响应组装设计

> 本文梳理 `/api/chat` 管道(server.py:2649-4926)里"重排序"和"最终响应组装"两层机制,以及 Android 客户端如何渲染。
> 所有结论均可回溯到具体行号;若代码改动,请同步更新本文。

---

## 1. 排序数据流总览

```
用户消息
  │
  ├─► [1] AI 意图识别 (QWEN_FAST_MODEL) → food_keywords / required_elements / sort_by / max_price / user_vibe
  │
  ├─► [2] SQL 候选查询 (ORDER BY rating DESC, review_count DESC) → 粗排候选池
  │
  ├─► [3] Python 层多重排序 (smart 打分 → 距离桶 → 价格重排 → 偏好软排)
  │
  ├─► [4] candidates[:15] 切片 → 打包成 candidate_text
  │
  ├─► [5] RANKING_PROMPT (QWEN_MODEL) 一次调用 → {greeting, picks, followup}
  │         ─ 排序顺序 (BEST→WORST)
  │         ─ 文案 (greeting + reason + followup)
  │
  └─► [6] 后端组装最终响应 {text, restaurants}
```

**核心设计**:Python 层排序只为决定"哪 15 家进入 AI 的可见窗口";最终顺序和文案由 AI 一次推理产出。

---

## 2. Python 层排序的 5 个节点

### 2.1 SQL 层返回顺序(server.py:1307-1327)

数据库直接 `ORDER BY rating DESC, review_count DESC`。

### 2.2 smart 打分 → 初排(server.py:1472-1478)

候选的 `relevance` 由 smart term 系统算出:

- `_smart_term_specificity` (server.py:1077) — term specificity = (10 if required else 0) + longest_phrase_length
- `_smart_required_groups` (server.py:1146) — 把"鸡肉/牛肉/虾"等归为语义组
- `_smart_food_terms` (server.py:1216) — 主函数,产出按 specificity 排序的 terms

初排 sort key:

```python
def _sort_key(x):
    rel = -x["relevance"]                              # 相关度优先
    dist_bucket = int(x.get("distance") or 99)        # 距离按 1km 分桶
    rating_score = -(rating * min(reviews+1, 200) / 200.0)  # 评分 × 评论数(封顶 200)
    return (rel, dist_bucket, rating_score)
```

**关键设计**:距离用 `int()` 分桶,避免 0.1km 微差异压过评分差距。

### 2.3 sort_by 分流(server.py:3636-3641)

根据 AI 意图里的 `sort_by` 字段(distance/price/rating)分别排序,都把 `has_menu=True` 排前。

### 2.4 菜单相关性验证(server.py:3686 附近)

`calculate_menu_relevance` (server.py:604) 扫描 `menu_sample`:
- 组合食物(鸡肉汉堡)必须所有 `required_elements` 都出现
- 按 `(-menu_relevance, -rating_score)` 重排

### 2.5 PRICE-SORT FIX — 真正的"硬重排"(server.py:4153-4177)

当用户要最便宜时,**重新查 DB 拿每个候选最便宜的匹配菜品真实价格**,再据此重排:

```python
if sort_by == "price" and candidates and food_kws:
    _sql = ("SELECT restaurant_id AS rid, MIN(price) AS mp FROM menu_items "
            "WHERE restaurant_id IN (...) AND price >= 25000 "
            "AND (LOWER(name) LIKE ? ...) GROUP BY restaurant_id")
    for c in candidates:
        c["_min_match_price"] = _minp.get(c["id"], 10**12)
    candidates.sort(key=lambda x: (0 if x.get("has_menu") else 1,
                                    x.get("_min_match_price", 10**12)))
```

**为什么需要这一步**:`price_hint` 是字符串解析(不可靠),会在 AI 看到的前 15 个候选中把真正最便宜的店挤出去。所以必须在切片前用真实菜单价格重排。注释原文在 server.py:4150-4152。

### 2.6 用户偏好软排(server.py:3977-4008)

`preferences` 字段里有 `distance first` / `under 50,000` / `cheap eats` / `healthy` 时,再做一轮偏好排序。

---

## 3. AI 最终排序与文案(server.py:4527-4553)

```python
system_prompt = RANKING_PROMPT
user_prompt += f"\nCandidate restaurants:\n{candidate_text}"
resp = await qwen_client.chat.completions.create(
    model=QWEN_MODEL, messages=msgs,
    temperature=0.85, top_p=0.92, max_tokens=1500,
    extra_body={"enable_thinking": False})
ai_text = resp.choices[0].message.content
```

**排序 + 文案在同一次调用产出**。AI 输出结构:

```json
{
  "greeting": "...",
  "picks": [{"index": 0, "reason": "...", "flex_tags": ["coffee"]}],
  "followup": "..."
}
```

RANKING_PROMPT 里同时定义了排序规则(DISTANCE / VALUE / BUDGET RULE)和文案风格(signature dish、appetite-triggering words)——这解释了为什么 prompt 既要讲顺序又要讲写法。

**Python 后处理(server.py:4554-4600)** 只做安全兜底:
- `_to_simplified_chinese` (繁→简,OpenCC t2s)
- greeting 超 150 字符按句号截断
- index 边界校验(丢掉 AI 编造的越界 pick)
- picks 不足 5 个时按候选原顺序补齐
- `format_response` 剥离货币换算(汇率不稳,一律只用越南盾)

---

## 4. 最终响应组装 {text, restaurants}

### 4.1 `text` 字段 — format_response(server.py:1484-1527)

把 Qwen 返回的 `{greeting, picks, followup}` + `candidates` 数组**拍平成纯文本字符串**:

```
{greeting}

1. {candidates[pick[0].index].name}
   {reason}
   📍 {address}
   ⭐ 4.8(1234) | 📏 1.2km 距你 | 🏪 grab
   💰 {price_hint}
   🎁 PROMO!
   {url}

2. ...

{followup}
```

**重要后果**:结构化的 picks/reason/followup 在客户端这一层**不存在**,菜品名只是嵌在自然语言句子里的普通字符。

### 4.2 `restaurants` 字段 — 后端在响应阶段组装(server.py:4601-4700+)

**完全由后端组装**:根据 AI 给的 picks 反查数据库,再附加补充字段。

```python
for p in picks:
    idx = p.get("index", 0)
    r = db2.execute("SELECT * FROM restaurants WHERE id = ?",
                    (candidates[idx]["id"],)).fetchone()
    rd = dict(r)
    rd["reason"] = p.get("reason", "")                    # 来自 AI
    rd["distance_km"] = round(candidates[idx]["distance"], 1)
    rd["avg_price"], rd["min_price"] = ...                # 现查 menu_items 聚合
    rd["sample_dishes"] = candidates[idx]["menu_sample"][:3] or ...
    rd["tags"] = [...]                                     # 相对排名标签
```

| `restaurants[]` 字段 | 来源 |
|---|---|
| `id`、`name`、`platform`、`cuisine`、`rating`、`review_count`、`address`、`url` | DB 查出的完整 `restaurants` 行(server.py:4638) |
| `reason` | AI picks[].reason(server.py:4641) |
| `distance_km` | 候选阶段算好的距离(server.py:4643) |
| `avg_price`、`min_price` | 响应阶段现查 menu_items 聚合(server.py:4646-4651) |
| `sample_dishes` | 候选阶段缓存的 menu_sample[:3] 或现查(server.py:4654-4661) |
| `tags`(nearby/value/popular/fresh) | 响应阶段基于这批 picks 的相对排名(server.py:4605-4675) |

### 4.3 相对排名标签(server.py:4605-4675)

针对这批 picks 内部算各维度排名,让 5 张卡片各有侧重不雷同:

- `nearby` — 距离升序前 50%
- `value` — 价格升序前 50%
- `popular` — 评论数过区域均线且评分 ≥ 4.6 的前 50%
- `fresh` — 评分 ≥ 4.7 但评论数低于均线(`REGION_REVIEW_P20 ≤ reviews < REGION_AVG_REVIEWS`)

### 4.4 最终返回(server.py:4892)

```python
_resp_val = {"text": full_text, "restaurants": picked if picked else None}
```

**注意**:多种 fallback/错误路径(menu_query、followup、place fallback,见 server.py:3187/3300/3329/3967)下 `restaurants` 为 `None`,只有 `text`——卡片只在主推荐路径下出现。

---

## 5. Android 客户端渲染现状

### 5.1 数据模型(`ApiModels.kt:162-165`)

```kotlin
data class ChatResponse(
    @SerializedName("text") val text: String = "",
    @SerializedName("restaurants") val restaurants: List<RestaurantDto>? = null,
)
```

### 5.2 渲染能力清单

| 内容 | 渲染方式 | 可点击 |
|---|---|---|
| `text`(含 greeting+reason+followup) | 普通 `Text`(`ChatBubbles.kt:104-128` `AiTextBubble`) | 否 |
| `text`(在 AiRestaurantsBubble 里) | 普通 `Text`(`ChatBubbles.kt:133-238`) | 否 |
| `restaurants[]` 卡片 | 卡片 UI(`ChatBubbles.kt:315-668`) | 整卡片 → `MerchantDetailScreen`(`WansoNavGraph.kt:294-319`) |
| Order 按钮 | 按钮(`ChatBubbles.kt:653`) | 跳外链 Grab/ShopeeFood |

### 5.3 没实现的能力

- `reason` 文本里的菜名(如"招牌会安鸡饭")**不可点击** — 点了没反应
- `greeting` 不可点击
- 没有 Markdown 渲染(无 commonmark / markwon)
- 没有 `AnnotatedString` / `ClickableText` / `SpannableString` / `Linkify`
- 不能从 reason 直接跳到具体菜品(只能整张卡片跳到餐厅详情)

### 5.4 若要加"菜品可点击",需改两处

1. **后端 `format_response`** 别再把 picks 拍平成纯文本 — 直接返回 `{greeting, picks:[{index, reason, dish_refs:[...]}], restaurants:[...]}`,让客户端能拿到结构。
2. **Android 端** 用 `AnnotatedString` + `ClickableText`(或 Compose 的 `clickable` span)渲染 reason,把菜名 span 绑定点击事件跳到 `MerchantDetailScreen` 的菜品锚点。

当前 `text` 字段的存在本身就说明这个 API 是为"文本聊天气泡 + 旁挂餐厅卡片"的简单模型设计的。

---

## 6. 关键文件索引

| 模块 | 路径 | 行号 |
|---|---|---|
| chat 管道主函数 | `server.py` | 2649-4926 |
| smart 打分系统 | `server.py` | 1048-1300 |
| `_smart_food_terms` 主函数 | `server.py` | 1216 |
| 候选 SQL 查询 | `server.py` | 1307-1327 |
| 初排 `_sort_key` | `server.py` | 1472-1478 |
| `format_response`(拍平为 text) | `server.py` | 1484-1527 |
| `calculate_menu_relevance` | `server.py` | 604 |
| PRICE-SORT FIX | `server.py` | 4153-4177 |
| 偏好软排 | `server.py` | 3977-4008 |
| RANKING_PROMPT 定义 | `server.py` | 706 |
| AI 调用入口 | `server.py` | 4527-4553 |
| restaurants 字段组装 | `server.py` | 4601-4700 |
| 相对排名标签 | `server.py` | 4605-4675 |
| 最终返回 `_resp_val` | `server.py` | 4892 |
| 排序提示词(中文版) | `docs/zh/排序提示词v0614.txt` | — |
| 意图识别提示词(中文版) | `docs/zh/意图识别提示词v0614.txt` | — |
| Android ChatResponse | `my-app-android/.../ApiModels.kt` | 162-165 |
| Android 聊天气泡 | `my-app-android/.../search/ChatBubbles.kt` | 104-668 |
| Android 导航 | `my-app-android/.../navigation/WansoNavGraph.kt` | 294-319 |
