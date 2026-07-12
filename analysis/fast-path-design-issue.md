# Fast path 跳过 LLM 排序的设计缺陷分析

> 分析对象:`server.py` 中 `_strict_food_chat_response` / `_smart_food_terms` / `_wanso_food_semantic_override` 触发的 strict food fast path
> 分析日期:2026-06-26
> 配套阅读:`docs/analysis/recommendation-analysis.md`、`docs/analysis/picks-insufficient-5.md`、`wanso-backend/CLAUDE.md`
> 问题陈述:对同一语义的"想吃牛肉"请求,ZH 输入走 fast path 返回死板模板,VI 输入走完整 AI pipeline 返回高质量推荐文案,体验割裂。进一步审查发现 fast path 对**组合菜**和**单宽泛食材误塞 required_elements** 两种情况都跳过了本应执行的 LLM 排序/生成。

---

## 一、结论(先说判断)

当前 fast path 设计存在两类问题:

1. **设计层面**:组合菜(beef pho / chicken burger 等)命中 `_SMART_FOOD_ALIASES` 或带 `req_groups` 的 `_smart_food_terms` 时,完全跳过 LLM 排序,直接 Python sort + f-string 模板返回。Python sort 只能按价格/评分/平台字段排,排不出**招牌菜点评、场景匹配、多样性合理度**等需要语义理解的东西。

2. **健壮性层面**:`_smart_food_terms` 的"宽泛词过滤"(`too_broad_single`)依赖 intent 模型严格遵守"单食材 → `required_elements=[]`"的 prompt 规则。模型在 ZH 输入上偶尔违规,把单食材塞进 `required_elements`,触发 `req_groups` 旁路,导致**单宽泛食材查询**也错误地走了 fast path。

**fast path 应该保留的场景**:仅限 `_strict_food_terms` 硬编码的 3 道国民菜(phở / bánh mì / bún chả)。这类查询用户要的是"附近所有 phở 按评分排序",不需要 AI 主观推荐,且查询量大,值得优化。

**fast path 不应该处理的场景**:组合菜、单宽泛食材。这些场景 SQL 召回只是第一步,后续该由 LLM 在候选上做 re-rank + 文案生成。

---

## 二、现状对照(同一个"想吃牛肉",两种命运)

### Case 10 (ZH): `今晚想吃牛肉,有什么推荐`

intent 模型在 ZH 上违反 prompt 规则,返回非空 `required_elements`(推测含 `"bò"` 或 `"牛肉"`)。导致:

1. `_smart_required_groups` (server.py:1085) 产出 `req_groups = [["bo","bò","beef"]]`
2. `_smart_food_terms` 的宽泛词过滤被旁路 (server.py:1195 `if too_broad_single and not req_groups: continue` 不再触发)
3. `terms` 非空 → `_strict_food_chat_response_base` 不返回 None (server.py:6453)
4. server.py:7115 fast path gate 命中 → 跳过 QWEN_MODEL 调用 → f-string 模板返回

返回示例(模板):
```
我找到了这些和 bò 直接相关的选项。下面价格是菜单菜品价...
1. Phở Dịu [GrabFood] — phở tái nạm,125000đ [id:grab-5-C6LHCK6HHBE1NA]
2. ...
```

### Case 11 (VI): `tôi muốn ăn bò`

intent 模型在 VI 上遵守规则,返回 `required_elements: []`。导致:

1. `req_groups = []`
2. `_smart_food_terms` 把 `"bò"` 当 `too_broad_single` 过滤 → `terms = []`
3. `_strict_food_chat_response_base` 在 server.py:6453 返回 None
4. fast path 不触发 → 完整 AI pipeline → QWEN_MODEL 生成

返回示例(AI 生成):
```
Thèm thịt bò là phải ăn ngay cho đã! 🥩 Phở Dịu gần bạn nhất,
tô phở tái nạm nước dùng ngọt lắm chỉ 125k, đảm bảo làm dịu cơn đói
tức thì. Cùng khám phá thêm vài quán bò ngon khác nhé!

1. Phở Dịu - Phở Bò & Phở Dê - Nguyễn Bỉnh Khiêm [id:grab-5-...]
2. Cơm Gà, Gà Lắc Phô Mai, Mì Ý Sốt Bò Bằm: Bếp Kim Dung [id:grab-5-...]
3. Lẩu Bò Triều Châu Lĩnh Ngưu 领牛潮汕牛肉火锅 [id:sf-1260003]
4. ...
```

**两种回复质量差异巨大**(emoji、单菜点评、招牌菜前缀、收尾节奏词都只在 AI 路径里出现),且差异完全由**输入语言**决定,用户无法预测。

---

## 三、设计原意 vs 实际行为

### fast path 的设计取舍

`_strict_food_chat_response` (server.py:6410-6752) 跳过 LLM 的代价是:
- ✅ 成本省:零 QWEN_MODEL 调用
- ✅ 速度快:纯 SQL + Python
- ✅ 确定性强:同输入同输出,便于回归
- ✅ 反幻觉:不靠模型编价格/菜名
- ❌ 文案死板:f-string 拼接,无个性化
- ❌ 无语义排序:Python sort 只能按字段排
- ❌ 无场景理解:不知道用户是夜宵还是正餐

设计者把 fast path 用于高频且"用户要的就是列表"的场景,这本身合理。问题在于**适用边界划得过宽**。

### 当前 fast path 触发条件总览

| 命中来源 | 触发条件 | 用户意图明确度 | 设计是否合理 |
|---|---|---|---|
| `_wanso_food_semantic_override` (server.py:6316) | phở / bánh mì 的精确 pattern | ✅ 极明确 | ✅ 合理 |
| `_strict_food_terms` (server.py:795) | 硬编码 phở / bánh mì / bún chả | ✅ 极明确 | ✅ 合理 |
| `_SMART_FOOD_ALIASES` (server.py:945) | 别名命中(如 "beef pho") | ✅ 明确 | ⚠️ 候选质量差异大,该 LLM re-rank |
| `_smart_food_terms` + `req_groups` | 组合菜 / 模型违规 | ⚠️ 视情况 | ❌ 不合理 |

### `req_groups` 旁路口的设计意图

`_smart_food_terms` 在 server.py:1191-1210 的逻辑:

```python
too_broad_single = nk in {
    "ga", "gà", "bo", "bò", "heo", "tom", "tôm",
    "rice", "com", "cơm", "bun", "bún", "mi", "mì",
    "sweet", "ngot", "ngọt", "cay", "spicy"
}
if too_broad_single and not req_groups:   # ← 单宽泛词 + 没组合 → 跳过, 交 AI
    continue
if (" " in nk) or req_groups:             # ← 多词组合 或 有 required → 收
    terms.append({...}); break
```

设计意图(读 intent prompt server.py:6993-7000 推断): `required_elements` 仅在组合菜时非空(prompt 明文规定 "Single foods → []")。所以 `req_groups` 非空 ⇒ 用户问的是组合菜 ⇒ 即使用户写的词单看是 "bò" 这种宽泛词,作为组合菜的一个组件,整体也足够精确,可以走 fast path。

**问题**: 这套逻辑把 intent 模型当作可信源。模型一旦违规(把单食材塞进 required_elements),整条推理链崩塌。

---

## 四、Python sort 排不出来的东西

fast path 的排序逻辑 (server.py:6560-6627):

```python
# compare 模式
enriched.sort(key=lambda r: (
    _price_num(r.get("estimated_total"), 10**12),  # 价格升序
    0 if r.get("platform") == "shopee" else 1,      # 平台优先
    -_price_num(r.get("rating"), 0)                 # 评分降序
))

# 普通模式: 按 SQL 候选返回顺序 + 强制双平台均衡(塞一个 grab + 一个 shopee)
```

它排不出:

1. **某家店的某道菜是不是招牌** —— SQL 只能确认菜单里有"phở bò",但这家可能主推"bún bò",phở bò 只是凑数
2. **某道菜的真实水平** —— `rating` 是全店总评,不代表这道菜
3. **多样性合理度** —— 排前 5 全是同一家连锁的不同分店,Python sort 看不出问题
4. **场景匹配** —— "今晚想吃牛肉"是想要正餐还是夜宵?用户语气里有线索,Python sort 看不到

case 11 那条 AI 回复里 `"Phở Dịu gần bạn nhất, tô phở tái nạm nước dùng ngọt lắm"` 这种**单菜点评**,fast path 永远写不出来。

---

## 五、修复方向(按改动量排序)

### 方向 A:最小改动 — 收紧 `req_groups` 旁路

在 `_smart_food_terms` 加一条:`req_groups` 长度 < 2 时,视为单食材,不旁路过滤。

```python
# server.py:1195 当前
if too_broad_single and not req_groups:
    continue

# 改为
if too_broad_single and len(req_groups) < 2:
    continue
```

- ✅ 修掉"单食材误塞 required_elements"导致的 case 10
- ❌ 不解决组合菜该不该走 fast path 的根本问题
- ❌ 副作用: 真正的"单组件 required_elements"(罕见,prompt 不鼓励)也被卡

### 方向 B:中等改动 — 仅硬编码菜走 fast path,组合菜回 AI

收紧 fast path gate,只让 `_strict_food_terms` + `_wanso_food_semantic_override` 命中时触发:

```python
# server.py:7115 当前
if intent.get("intent") in ("search", "compare") and not intent.get("target_restaurant"):
    _strict_food_resp = _strict_food_chat_response(...)
    if _strict_food_resp is not None:
        ...
        return _strict_food_resp

# 改为: 让 _strict_food_chat_response 内部 _smart_food_terms 不再产出 terms,
#        组合菜自动落到 AI pipeline
```

具体做法: 在 `_strict_food_chat_response_base` 的 server.py:6446 `_merge_food_terms` 调用里,去掉 `_smart_food_terms` 那一路,只保留 `_strict_food_terms`。组合菜(beef pho / chicken burger)自动落到 AI pipeline,AI 自己根据 `food_keywords` 召回+排序。

- ✅ 修掉 case 10
- ✅ 组合菜得到 AI 排序,文案质量提升
- ❌ 副作用: 缓存 key 不变但响应内容变,线上需清 30 分钟响应缓存或等过期
- ❌ 副作用: 组合菜查询从 0 次模型调用变成 1-2 次,成本上升

### 方向 C:大改 — 引入"SQL 召回 + LLM re-rank"中间档

复用现有 AI pipeline,但在 prompt 里把 SQL 候选作为 context 喂给 QWEN_MODEL:

```
查询类型                          │ 处理方式
─────────────────────────────────┼──────────────────────────────────
phở / bánh mì / bún chả (硬编码3) │ fast path (SQL + sort + 模板)
─────────────────────────────────┼──────────────────────────────────
组合菜 (beef pho, chicken burger) │ AI pipeline,SQL 收紧召回 → LLM 排序生成
─────────────────────────────────┼──────────────────────────────────
单食材 (bò, gà, 烧烤)             │ 完整 AI pipeline (现状不变)
```

- ✅ 三档各司其职,质量最好
- ❌ 改动面大:要改 prompt、改 candidate 注入逻辑、改测试
- ❌ 需要重新跑 chat_smoke 校验所有 case

### 推荐路径

先做 **方向 B**(收紧到仅硬编码菜走 fast path),理由:

1. 一行改动解决 80% 问题
2. phở / bánh mì / bún chả 这 3 道高频菜继续享受 fast path 优化,成本可控
3. 组合菜在 AI pipeline 里现有 prompt 已经能处理(`food_keywords` 足够召回)
4. 方向 A 的 `len(req_groups) < 2` 是治标,但组合菜该走 AI 这个根本问题没解,后续还会被模型其它违规输出绕过

---

## 六、验证清单(任意方向改完后都要跑)

| Case | 输入 | 期望路径 | 验证方式 |
|---|---|---|---|
| 1 | EN - phở | fast path (硬编码) | docker logs 无 QWEN_MODEL 调用 |
| 2 | ZH - 越南法包 | AI pipeline | docker logs 有 intent + QWEN_MODEL |
| 3 | VI - bún chả | fast path (硬编码) | docker logs 无 QWEN_MODEL 调用 |
| 10 | ZH - 牛肉 | AI pipeline | docker logs 有 intent + QWEN_MODEL |
| 11 | VI - bò | AI pipeline (现状不变) | docker logs 有 intent + QWEN_MODEL |
| 6 | phở 比价 | fast path (硬编码 + compare 分支) | ⚠️ 会写 refresh_jobs,慎跑 |

辅助检查:

```bash
# 看每次请求实际命中哪个分支
docker logs wanso-backend-1 2>&1 | grep -E "INTENT_FASTPATH|STRICT_FOOD|QWEN_MODEL"

# 看 intent 模型 raw 输出,确认 required_elements
docker logs wanso-backend-1 2>&1 | grep -A1 "intent"
```

---

## 七、相关代码索引

| 功能 | 位置 |
|---|---|
| Fast path gate | `server.py:7115` |
| Local Intent Fastpath (intent 模型前的预检) | `server.py:6917-6939` |
| `_wanso_food_semantic_override`(只识 phở / bánh mì) | `server.py:6316-6390` |
| `_strict_food_terms`(硬编码 phở / bánh mì / bún chả) | `server.py:795-849` |
| `_SMART_FOOD_ALIASES`(组合菜别名表) | `server.py:945-984` |
| `_smart_food_terms` + `too_broad_single` 过滤 | `server.py:1155-1212` |
| `_smart_required_groups`(required_elements → groups) | `server.py:1085-1114` |
| `KEYWORD_MAP`(通用扩展,不门 fast path) | `server.py:445-498` |
| Fast path 主体 | `server.py:6410-6752` |
| Python sort (普通模式) | `server.py:6594-6627` |
| Python sort (compare 模式) | `server.py:6560-6593` |
| Intent prompt | `server.py:6949-7010` |
| intent 模型调用 + fallback | `server.py:7077-7087` |
