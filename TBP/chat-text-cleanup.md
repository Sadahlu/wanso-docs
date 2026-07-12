# /api/chat 响应字段治理:greeting 截断 + text 字段污染

## 问题

`/api/chat` 的 `text` 字段当前承担了两个本不该混在一起的职责:

1. 给用户看的自然语言回复
2. 给后端下一轮会话状态恢复用的结构化标记 `[id:xxx]`

由此派生出两个具体问题:

### 问题 A:greeting 被 150 字符截断 ✅ 已修 (2026-06-24)

`server.py:8962-8973`(原)有一段"过 150 字符时找最后一个句末标点截断"的逻辑,而且
标点列表 `["。","！","!",".","？","?",","]` 是顺序匹配 + 命中即 break —— 多句子
greeting 会在前 150 字符内**第一个英文 `!`** 处被砍掉。冒烟用例 5 "recommend a brunch
spot" 的 greeting 被截到 "spot!" 就是这条逻辑干的。

此外 `_stream_recommend_ai` 流式推送也有 `GREETING_LIMIT = 150` 的截断
(`server.py:2384` + `2420-2421`),导致**流式增量推到 ~150 字符就停,done 帧再用
标点截断算法二次砍**,两处截断点不一致。

**根因**:text 字段在做本该前端 UI 做的事(控制显示长度)。

**修复**:删掉 `server.py:8962-8973` 的标点截断 if 块,删掉 `_stream_recommend_ai`
里 `GREETING_LIMIT = 150` 定义和对应截断 if。完整 greeting 返回给前端,前端 UI 自己
处理气泡高度/换行/"展开全文"。

### 问题 B:text 拼装 `[id:xxx]` 暴露内部标记给用户 ❌ 待修

后端把餐厅 id 直接拼进 text:

- strict food fast path:`server.py:6854/6869/6884/6923/6931/6939`
- AI 正常生成路径:`server.py:9314`(格式 `f"{i+1}. {p['name']} [id:{p['id']}]"`)

下一轮 chat 解析时(`server.py:7089-7102`)又用 regex 从 assistant 的 content 字符串
里把这些 id 抓回来恢复 `conv_state["shown_restaurants"]`:

```python
if "[id:" in c:
    ids_names = re.findall(r'(\d+)\.\s*(.+?)\s*\[id:([^\]]+)\]', c)
```

**问题**:

- 后端拼在 text 里的"编号列表 + [id:xxx]" 实际**对用户完全无用** —— 客户端整行
  过滤掉(见下文客户端调研)
- `/api/feedback` 也用同样套路从 text regex 抓餐厅(`server.py:9390-9405`),text 没
  有编号列表时反馈和餐厅就关联不上
- `restaurants` 数组**已经在响应里有 id**(`server.py:9320` `"restaurants": picked`),
  只是没作为结构化字段贯穿对话历史,只能"打捞"

**根因**:早期对话历史模型只存 text 字符串、不存结构化 restaurants 数组,后端被迫从
字符串里 regex 抓信息,内部标记污染了展示字段。

## 客户端调研结果 (2026-06-24)

客户端位置:`F:\Wanso\my-app-android`(独立 repo,不在 wanso-backend 的 monorepo)。

### 关键发现 1:用户实际看不到 `[id:xxx]`,但也看不到编号列表

客户端 **3 处**用同一个正则 `^\d+\.\s+.+\[id:[^\]]+]\s*$` **整行删除**带 id 的列表行:

- `SearchResultsScreen.kt:224-230`(auto-refetch 场景)
- `SearchResultsScreen.kt:469-477`(主 chat 请求)
- `RestaurantRepository.kt:190-198`(chat 函数)

所以用户:

- **看不到 `[id:xxx]`**(被整行过滤)
- **也看不到 `1./2./3./4.` 这种编号列表**(整行被删)
- 实际看到的是 greeting + 横滑卡片(`AiRestaurantsBubble`,ChatBubbles.kt:133-274)

### 关键发现 2:客户端不区分后端路径

过滤逻辑基于**响应字段**(text / restaurants),不是基于响应来自哪条 fast path /
cache / AI。所有路径(AI / strict fast path / constraint fast path / cache 命中)都走
同一个 SSE done 帧处理。

| 后端路径 | text 里的列表行格式 | 客户端处理 |
|---|---|---|
| AI 生成 | `1. The Running Bean ... [id:sf-1275359]` | 整行删 |
| Strict fast path | `1. THE TAJ ... [GrabFood] — , N/A [id:grab-5-...]` | 整行删 |
| Constraint fast path | (text 里**本来就没拼** id) | N/A |
| Cache 命中 | 复用上次拼好的 text | 同样过滤 |

### 关键发现 3:协议冗余 —— 客户端被迫再拼一次 `[id:xxx]`

`SearchResultsScreen.kt:306-328` 为了配合后端 regex 解析,**自己又把 restaurants 数组
重新拼成 `"1. name [id:xxx]"` 文本**作为 history.content 发回后端。完整链路:

```
后端 text (含 [id:xxx])
    ↓
客户端过滤整行 → 用户看 greeting + 卡片
    ↓
客户端存 history 时,【重新拼成】"1. name [id:xxx]" 当 content
    ↓
下轮请求发回后端
    ↓
后端 regex 抓 id 恢复 shown_restaurants
```

也就是说,客户端为了配合后端的 regex,自己要再拼一次 `[id:xxx]`。这是协议冗余且脆弱
的根源 —— 改造后两边代码都更干净。

### 关键发现 4:流式体验差异(改造后保留)

- **AI 路径**:chat_stream 推多个 `text_delta` 帧,用户看到 greeting 逐字打字 → 卡片
- **Fast path / Cache 命中**:chat_stream 不调 `_stream_callback`,只推 done 帧 →
  用户看到**直接出现** intro + 卡片(无打字动画)

Step 2 改造后这个差异保留 —— 体验不退化。

### 客户端改造工作量评估(很小)

| 字段 / 组件 | 现状 | 改造 |
|---|---|---|
| `ChatResponse.restaurants` | `ApiModels.kt:162-165` 已有 | 无 |
| `ChatHistoryMessage` | `ApiModels.kt:146-160` 简化为 `{role, content}` | **加 `restaurants` 字段** |
| `ChatHistoryStore` | `ChatHistoryStore.kt:25-36` 已存 `restaurantsJson` | 无 |
| `ChatMessageDto`(跨设备同步) | `ChatHistoryStore.kt:244-248` 已支持 `restaurants` | 无 |
| UI 过滤 | 3 处正则 | 改造完成后可删 |

**真正需要改的就 2 处**:

1. `ApiModels.kt:146-160` `ChatHistoryMessage` 加 `restaurants: List<...>? = null`
2. 删 `SearchResultsScreen.kt:306-328` 的"restaurants 拼回 raw 文本"逻辑

## 修复计划

### Step 1:删 greeting 截断 ✅ 已做 (2026-06-24)

- 删 `server.py:8962-8973` 标点截断 if 块
- 删 `_stream_recommend_ai` 里 `GREETING_LIMIT = 150` 定义和对应截断 if
- 不依赖客户端改动,可独立部署

### Step 2:text 去 `[id:xxx]` 拼装 + history 协议改造 ❌ 待做

需要前后端协调,breaking change。基于客户端调研结果细化方案:

**后端**(wanso-backend/server.py):

1. **后端先行(兼容期)**:`server.py:7089` 加 fallback,优先读 `h.get("restaurants")`,
   text regex 作为兼容路径保留
2. **客户端跟进**(见下)
3. **后端 cleanup(灰度后)**:去掉 text 拼装 `[id:xxx]`
   - 6 处 strict fast path:`server.py:6854/6869/6884/6923/6931/6939`
   - AI 正常路径:`server.py:9314`
   - 同时去掉 regex 解析:`server.py:7089-7102`
4. **顺手解决漏拼路径**:`server.py:9366`(AI 异常 fallback)和 `server.py:10454`
   (constraint fast path)当前就没拼 `[id:xxx]` —— 改造后用结构化字段统一,**不再
   需要单独补**
5. **`/api/feedback` 同步**:`server.py:9390-9405` 改读 history 的 restaurants 字段

**客户端**(my-app-android):

1. `ApiModels.kt:146-160` `ChatHistoryMessage` 加 `restaurants: List<...>? = null`
2. 删 `SearchResultsScreen.kt:306-328` 的"restaurants 拼回 raw 文本"逻辑,改成直接发
   结构化对象
3. 删 3 处正则过滤(`SearchResultsScreen.kt:224-230/469-477` +
   `RestaurantRepository.kt:190-198`)—— 后端去掉 text 里的 `[id:xxx]` 后这些过滤
   无用,删掉代码更干净(也可以保留作兼容,但没必要)
4. 灰度后视情况清理

**目标 history item 结构**:

```python
{
    "role": "assistant",
    "content": "Craving a relaxed brunch? ...",   # 纯文本,无 [id:xxx]
    "restaurants": [                               # 结构化,独立字段
        {"id": "sf-1275359", "name": "The Running Bean ...", "position": 1},
        ...
    ]
}
```

## 待确认

- ~~客户端代码是否在同一个 monorepo、是否能协调改动~~ ✅(2026-06-24)独立 repo
  `F:\Wanso\my-app-android`,可改,工作量小(2 处核心改动)
- 灰度过渡期多长(决定兼容路径保留多久)

## 触发本次讨论的来源

- 冒烟用例 5 SSE 流式输出观察到 greeting 在 "spot!" 截断
- 服务端日志 `WANSO_QWEN_CACHE_USAGE kind=ranking` 条目 `prompt_tokens=0`
  `completion_tokens=0` 是流式调用的副作用,不是没调用 —— 实际 3 次 Qwen
  调用(intent / other / ranking)都发生了
