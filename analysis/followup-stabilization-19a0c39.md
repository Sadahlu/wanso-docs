# 提交 `19a0c39` 分析:follow-up 推荐稳定化重构

> 对象:`wanso-backend` 仓库提交 `19a0c39 fix: stabilize follow-up restaurant recommendations`
> 作者:Wanso Backend <backend@wanso.ai>
> 时间:2026-06-28 03:41 UTC
> 分析日期:2026-06-28

---

## 一、一句话结论

提交标题是"稳定追问餐厅推荐",但**实际体量远超标题**:server.py 从 ~12k 行膨胀到 ~24k 行(净增 ~12k),内含 **33 个 `WANSO_*` 版本标记**(V1→V33,加多个子版本)。这是一次把本地候选文件(`server.py.wanso_*`)中积累的多版本迭代**批量合并进主线**的重构,核心是引入 `_wanso_finalize_chat_response_for_app` 作为 chat 响应的统一最终加工/守卫层。

---

## 二、文件改动总览

| 文件 | 改动 | 性质 |
|---|---|---|
| `server.py` | +22382 / −10084 行 | 大规模重写 + 重排 |
| `auth.py` | 1 行 | bug 修复 |
| `.gitignore` | +13 行 | 新增本地候选/secrets 排除 |

---

## 三、核心语义变更

### 1. 新增 chat 响应的"最终加工守卫层"(V1→V8 反复重定义)

`_wanso_finalize_chat_response_for_app` 是本次提交的核心。该函数在文件中被**多次重定义**(V1、V2、V6、V8 等),每次重定义都用 `_wanso_finalize_chat_response_for_app_before_v6` / `_before_v8` 等别名保留旧版作为兜底。这是稳定 follow-up 推荐的关键 —— 在响应返回给客户端之前统一执行过滤 + 字段归一化 + 候选补全。

### 2. 营业/删除状态过滤(V1)

新增 `_wanso_is_open_restaurant_for_response` + `_wanso_filter_open_restaurants_for_response`,确保不再把已关店或被删除的餐厅塞回给客户端。

### 3. "更多餐厅"追问流式补全(V14H / V14P2)

当用户问"还有吗 / 换一家"时,新路径 `_wanso_backfill_menu_search_open_restaurants` 会从菜单搜索中**补足仍在营业的新候选**,并接入 SSE stream 回调。

### 4. 卡片字段归一化 + VND-only(V2 / V14P2)

价格统一成整数 VND、菜单统计字段统一。**移除了多币种转换**(`convert_vnd` / `format_price_with_conversion` 被删除)—— 注释说明"AI tends to misuse conversion in greeting",即 AI 在 greeting 里频繁误用汇率转换。

### 5. 严格菜单 + 菜系匹配(V4/V5/V7)

引入多语言、去重音(`_wanso_v4_strip_accents` 等)的菜单匹配器,替代旧的 `term_matches` / `get_menu_summary_for_query` 路径。

### 6. AI pipeline 强化(V9–V33)

涵盖:AI 食物词解析缓存、位置感知缓存守卫、AI pipeline 追踪 print、beef query 安全重写、AI pick 顺序冻结快照、最终重排等。

---

## 四、`WANSO_*` 版本标记路线图

按版本号梳理本次提交引入的演进轨迹:

| 区间 | 主题 |
|---|---|
| V1–V6 | 开店过滤、卡片字段归一化、菜单 backfill、严格菜单匹配 |
| V7–V11 | 口音无关匹配、匹配质量排序、AI 食物词缓存、位置感知缓存 |
| V12–V18 | VND-only 价格、share/feedback、more-restaurants 流式补全、最终重排 |
| V20–V33 | fastpath env 开关、AI pipeline 追踪、AI 排序交接、beef 查询重写、菜单证据优先、AI pick 冻结/权威顺序 |

完整标记清单(共 37 条):

```
WANSO_OPEN_RESTAURANT_RESPONSE_FILTER_V1
WANSO_CARD_FIELD_NORMALIZE_V1
WANSO_CARD_CONTRACT_OVERRIDE_SAFE_V2
WANSO_OPEN_MENU_BACKFILL_V1
WANSO_OPEN_MENU_BACKFILL_CITY_SAFE_V2
WANSO_STRICT_MENU_MATCH_V4
WANSO_FAST_MULTILINGUAL_MENU_INTENT_V1
WANSO_BRANCH_SAFE_MENU_DEDUPE_V1
WANSO_EXPANDED_FOOD_RESOLVER_V2
WANSO_STRICT_SEMANTIC_GATE_V3
WANSO_UNIVERSAL_STRICT_MENU_RESOLVER_V5
WANSO_UNIVERSAL_FINALIZER_GUARD_V6
WANSO_ACCENT_SAFE_MENU_MATCHER_V7
WANSO_MATCH_QUALITY_RANKING_V8
WANSO_AI_FOOD_RESOLVER_CACHE_V9
WANSO_LOCATION_AWARE_CACHE_GUARD_V10
WANSO_V38G_COMBINED_AI_HANDOFF_FINALIZER_PRESERVE
WANSO_V38H_VND_ONLY_PRICE_DISPLAY
WANSO_SHARE_FEEDBACK_V12F4_BACKEND
WANSO_BACKEND_STREAM_MORE_RESTAURANTS_V14P2
WANSO_BACKEND_MORE_RESTAURANTS_FOLLOWUP_V14H
WANSO_BACKEND_EXACT_MENU_FOLLOWUP_GUARD_V14O
WANSO_V11_REMOTE_FINAL_LOCATION_CACHE_QUALITY
WANSO_LOCATION_METADATA_FINALIZER_V12
WANSO_QUALITY_RESPONSE_GUARD_V13
WANSO_FAST_PAYLOAD_FINAL_QUALITY_V14
WANSO_FAST_MENU_RELEVANCE_RANKING_V15
WANSO_FAST_MENU_HARD_PARTITION_V16
WANSO_RECOMMENDATION_AI_FIRST_V17
WANSO_FINAL_OUTPUT_RERANK_V18
WANSO_FASTPATH_ENV_SWITCH_V20
WANSO_KEYWORD_SENTENCE_GUARD_V22
WANSO_AI_PIPELINE_TRACE_V23
WANSO_FORCE_AI_RANKING_HANDOFF_V25
WANSO_SAFE_BEEF_QUERY_REWRITE_V26B
WANSO_BROAD_BEEF_MAIN_PATH_FIX_V27
WANSO_MENU_EVIDENCE_FIRST_V29
WANSO_AI_PICK_ORDER_REASONS_V30
WANSO_FREEZE_AI_PICK_SNAPSHOT_V31
WANSO_AUTHORITATIVE_AI_PICK_ORDER_V32
WANSO_UNIFIED_MENU_EVIDENCE_V33
```

---

## 五、附带的小改

### `auth.py:619` —— bug 修复

```diff
-        cur.execute("DELETE FROM user_favorites WHERE user_id = ?", (user["id"],))
+        cur.execute("DELETE FROM user_favorites WHERE user_id = ?", (user["user_id"],))
```

`clear_all_favorites` 里 key 名错的 bug —— JWT 解码出的 user dict 字段是 `user_id` 而非 `id`,旧代码会抛 KeyError 或删除条件错位。

### `.gitignore` —— 新增排除规则

```gitignore
# Wanso local/runtime secrets and generated files
*.pem
*.key
*.crt
wanso.db
wanso.db-*
*.db-*
*.pyc
server.py.wanso_*
server.py.*candidate*.py
.wanso_*
```

`server.py.wanso_*` 和 `server.py.*candidate*.py` 这两条说明:**本次提交的 WANSO 改动是从本地候选文件合并进来的**,而非直接在 `server.py` 上线性开发。

---

## 六、结构风险与后续编辑注意事项

### ⚠️ 同名函数多次定义、靠顺序覆盖

`_wanso_finalize_chat_response_for_app` 等多个关键函数在文件中被**重复定义多次**。Python 的语义是"后定义覆盖前定义",再加上 `_before_v6` / `_before_v8` 别名做软兜底。这种写法带来三个风险:

1. **静态阅读容易踩坑** —— 编辑者读到中间某个废弃版本会以为是生效版。
2. **diff 工具失效** —— 任何后续对早期版本的"小修"可能被后面的重定义整体覆盖,改动看不见效果。
3. **回退困难** —— 一旦发现新版有问题,无法用 `git revert` 单个 commit 回退(因为本提交本身就把 N 个版本捆在一起)。

### 编辑该区域前的操作守则

- 改任何 `_wanso_*` 函数前,先 `grep -n "def <函数名>"` 把所有同名定义列出来,**取最下面那份为生效版**。
- 不要在中间的废弃版本里改逻辑 —— 改动会被覆盖,等于无效。
- 如果要"清理"重复定义,**整段删除废弃版本**(包括对应的 `_before_v*` 别名)而不是留着,但要先用 `grep -n` 确认没有其他位置在调用那个别名。
- 该函数族周围有大量 `print("WANSO_BACKEND_V14H_MORE_RESTAURANTS_FOLLOWUP ...")` 调试 print —— 改逻辑时不要顺手删,它们是线上排障的 trace 锚点(见 V23 `WANSO_AI_PIPELINE_TRACE`)。

---

## 七、与 CLAUDE.md 的对应关系

本次提交的改动与 `wanso-backend/CLAUDE.md` 中以下条目强相关:

- **Chat 流水线第 9 步**(`server.py:2649–4926`):候选筛选 + AI 响应生成。V14H/V14P2 的"更多餐厅"补全路径正是接在这附近。
- **"严格食物词短路"**(`_strict_food_terms`, server.py:856):V4/V5/V7 的菜单匹配器是该路径的延伸,但**没有替换** `_strict_food_terms` 本身 —— 两者共存,短路检测仍走旧逻辑,新匹配器主要服务 follow-up 补全。
- **"匹配文件已有的注释语言(中英混用)"**:本次提交大量新增 `# WANSO_*` 英文标记 + 中文行内注释,延续了这个习惯。

---

## 八、复核命令

如需复核本分析的结论,可用以下命令:

```bash
# 提交元信息
git show --stat 19a0c39

# 所有新增的 WANSO 版本标记
git show 19a0c39 -- server.py -w | grep -E "^\+# WANSO_"

# 所有新增的函数定义(627 条,含重复重定义)
git show 19a0c39 -- server.py -w | grep -E "^\+(def |async def )" | wc -l

# 查看生效的 _wanso_finalize_chat_response_for_app 定义位置
grep -n "def _wanso_finalize_chat_response_for_app" server.py
```
