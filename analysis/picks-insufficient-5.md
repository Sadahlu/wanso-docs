# 推荐结果不足 5 家的根因分析:候选层先于 picks 层被砍

> 分析对象:`server.py` 中 `/api/chat` 推荐流水线的候选/picks 数量链路
> 分析日期:2026-06-24
> 配套阅读:`docs/analysis/recommendation-analysis.md`、`wanso-backend/CLAUDE.md`
> 问题陈述:线上观察到推荐卡片经常只有 3 家,而非预期的 5 家。

---

## 一、结论(先说判断)

**根因不在 picks 兜底逻辑,而在更下游的 candidates 构建层。**

两条推荐路径主 chat 和 strict_food 各自都有"把 picks 补齐到 5"的逻辑,且实现是正确的。但**它们都依赖底层候选列表本身 ≥ 5**。一旦候选构建阶段就被各种过滤/阈值砍到 3 家以下,picks 兜底也补不出来 —— 因为没有东西可补。

也就是说:**picks 不足 5 是症状,候选不足 5 才是病根。**

---

## 二、两条路径及其 picks 兜底(均已正确实现)

### 路径 A:严格食物词快捷路径 `_strict_food_chat_response_base`

位置:`server.py:6605`

候选来源:
- `strict_food_candidates(limit=40)` (server.py:6656)
- 再合并 `_explicit_restaurant_sql_candidates` (server.py:6666)
- 经 `_filter_default_compare_dish_candidates` / `_filter_explicit_restaurant_candidates` 过滤 (server.py:6674-6675)

picks 构建(非比较模式,server.py:6789-6822):
1. 先按 enriched 顺序取前 3 家(`if len(picks) >= 3: break`,server.py:6800)
2. 若前 3 家缺 grab → 补 1 家 grab(server.py:6803)
3. 若前 3 家缺 shopee → 补 1 家 shopee(server.py:6809)
4. 最后循环补到 `limit=5`(server.py:6815-6822)

**关键:第 4 步的循环确实能把 picks 补到 5,前提是 `len(enriched) >= 5`。**

### 路径 B:主 chat 函数

候选链(server.py:8200-8340):
1. 菜单存在性筛选 → `with_menu + without_menu[:3]`(server.py:8206)
2. 通用兜底(关键词无结果时)(server.py:8212)
3. **FINAL CATEGORY GATE**:品类硬门,只放行真匹配的店(server.py:8282-8291)
4. **DISTANCE FILTER**:默认 6km 砍距离(server.py:8302)
5. **DISTANCE FLOOR**:若不足 5,从 `all_backup`(品类门后、距离过滤前的池)按距离升序补到 5(server.py:8316-8325)

picks 兜底(server.py:8970-8978):
```python
if not is_compare and len(picks) < 5:
    _picked = set(p["index"] for p in picks)
    for _c in candidates:
        if len(picks) >= 5:
            break
        if _c["index"] not in _picked:
            picks.append({"index": _c["index"], "reason": ""})
```

**关键:DISTANCE FLOOR 和 picks 兜底都依赖 `candidates` 池本身 ≥ 5。**

---

## 三、候选被砍到 < 5 的具体嫌疑点(按可能性排序)

### 嫌疑 1(最可能):`strict_food_candidates` 的 score 阈值 + bbox

位置:`server.py:1263`

- **score 阈值**(server.py:1333):`if score < 45: continue` —— 匹配弱的菜品-店对直接 skip。具体菜品词(如罕见地方菜)很容易整批低于 45 分被全剔。
- **bbox 锁死**(server.py:1290-1293):有 lat/lng 时强制 `±0.065°`(约 ±7km)方形。超出方形的店根本不进 candidates。
- **term_specificity 早停**(server.py:1281):已攒够 8 家后,弱 specificity 的 term 不再追加。一般不会卡,但极端情况下可能让某个平台的店缺失。

### 嫌疑 2:主 chat 的 FINAL CATEGORY GATE 把候选砍光

位置:`server.py:8282-8291`

```python
_on = [c for c in candidates if _on_category(c)]
if _on:
    candidates = _on
else:
    candidates = []   # 直接清空,触发后续 _nearby_* 兜底
```

- 品类门判据严苛:店名命中 OR cuisine 字段命中 OR 菜单命中 ≥ 3 条(server.py:8264-8281)。
- 如果某个品类在附近本来就只有 3-4 家真匹配的店,过门后 candidates 就只有 3-4 家。
- DISTANCE FLOOR 想补,但 `all_backup` 也是品类门**之后**的池(server.py:8301 在 8289 之后),**补不到品类外的店**。

### 嫌疑 3:DISTANCE FILTER 6km + DISTANCE FLOOR 池子也小

位置:`server.py:8294-8325`

- 默认 6km 砍距离,如果用户位置偏远,附近本来就没几家。
- DISTANCE FLOOR 只能从"品类符合但距离 > 6km"的店补 —— 如果这些店也没有,补不动。

### 嫌疑 4(可能性较低):enriched 阶段 `is_deleted` 过滤

位置:`server.py:6700`

```python
row = conn.execute(
    "SELECT * FROM restaurants WHERE COALESCE(is_deleted,0)=0 AND id=?",
    (c["id"],)
).fetchone()
if not row:
    continue
```

candidates 里有但 restaurants 表标 `is_deleted=1` 的店会被剔。如果 DB 状态不一致,可能让 enriched 缩水。但通常量不大。

---

## 四、排查路径(需要线上日志确认走哪条)

服务端 `print(...)` 输出大量调试信息,从日志关键字可以快速定位:

| 关键字 | 含义 | 位置 |
|---|---|---|
| `STRICT FOOD` / `strict_food_query` | 走了快捷路径 A | server.py:6605+ |
| `FINAL CATEGORY GATE: N -> M` | 品类门把候选从 N 砍到 M | server.py:8287 |
| `DISTANCE FILTER: N -> M` | 距离过滤砍候选 | server.py:8305 |
| `DISTANCE FLOOR: topped up to N` | 距离兜底补到 N 家 | server.py:8325 |
| `GENERIC FALLBACK: found N popular` | 关键词无结果走热门兜底 | server.py:8239 |
| `RELAXED FALLBACK: by-food -> N` | 主 chat 兜底走 `_nearby_by_food` | server.py:8344 |
| `SMART FALLBACK: top-rated -> N` | 主 chat 兜底走 `_nearby_top_rated` | server.py:8350 |
| `PICKS TOPPED UP to N` | picks 兜底补到 N | server.py:8978 |
| `NEARBY-FOOD: term=... -> N in-box, kept M` | `_nearby_by_food` 内部计数 | server.py:2723 |

**最直接的诊断**:复现一次"只有 3 家"的请求,grep 这些关键字。如果看到 `FINAL CATEGORY GATE: 8 -> 3` 或 `DISTANCE FILTER: 6 -> 3` 且没有 `DISTANCE FLOOR: topped up to 5`,就是嫌疑 2/3 坐实。如果走的是 strict_food 路径且 enriched 里只有 3 家,就是嫌疑 1。

---

## 五、可能的修复方向(待日志确认根因后再选)

不要在没确认根因前盲改。下面是几个候选方向,各有副作用:

### 方向 A:放宽 `strict_food_candidates` 的 score 阈值

`server.py:1333` 的 `score < 45` 降到 35 或 30。**风险**:可能放回一批弱匹配/同名的店,污染推荐质量。需要看真实数据里 35-45 分段的店是不是用户想看到的。

### 方向 B:扩大 `strict_food_candidates` 的 bbox

`server.py:1290` 的 `±0.065°` 放宽到 `±0.09°`(约 10km)。**风险**:推荐的店更远,与 prompt 里"first 3 picks should be from the closer half by distance"的原则冲突。

### 方向 C:DISTANCE FLOOR 跨品类补齐

在 server.py:8316 的 DISTANCE FLOOR 之前,如果品类门后的候选 < 5,**临时**从品类门前的池里按距离取最近的补齐。**风险**:可能让"我要 phở"的结果里混进一家卖汉堡的(虽然近)。需要在 reason 里标注"非严格匹配"。

### 方向 D:`_nearby_by_food` / `_nearby_top_rated` 触发条件放宽

当前 server.py:8338 的兜底条件是 `if not candidates` —— 候选**完全为空**才触发。改成 `if len(candidates) < 5`,候选不足 5 就主动拉附近高分店补齐。**风险**:可能让具体菜品请求里混进不卖这道菜的店。需要 `_nearby_by_food` 优先,只有它返回 < 5 才用 `_nearby_top_rated` 补。

### 方向 E(最保守):只在 picks 层做"少于 5 不强求"

什么都不改 candidates 链,接受候选不足时返回少于 5 家,但在 AI prompt 里明确"如果候选不足 5,如实告诉用户附近这个品类选择有限"。**现状已经基本是这个行为**(prompt server.py:720、762 都有 "If fewer than 5 candidates truly match, return fewer")。这条算 baseline,不算修复。

---

## 六、附:相关行号速查

| 关注点 | 位置 |
|---|---|
| 严格食物词候选构建 | `server.py:1263` |
| 严格路径 picks 构建 + 补齐 | `server.py:6768-6822` |
| 主 chat 候选合并/dedup | `server.py:8327-8335` |
| 主 chat `_nearby_by_food` 兜底 | `server.py:8338-8344` |
| 主 chat `_nearby_top_rated` 兜底 | `server.py:8346-8350` |
| 主 chat FINAL CATEGORY GATE | `server.py:8244-8291` |
| 主 chat DISTANCE FILTER + FLOOR | `server.py:8294-8325` |
| 主 chat picks 补齐到 5 | `server.py:8966-8978` |
| `_nearby_by_food` 实现(内部 `tmp[:12]`) | `server.py:2668-2735` |
| `_nearby_top_rated` 实现(内部 `out[:15]`) | `server.py:2738-2766` |
| AI prompt 里 "Return exactly 5 picks" | `server.py:762` |
