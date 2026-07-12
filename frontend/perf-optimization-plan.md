# 客户端性能优化计划

> 范围：仅 Android 客户端 (`com.wanso.app.v2`)，不含后端
> 日期：2026-06-11
> 状态：待评审 / 未实施

## 背景

基于一次代码扫描（网络层、Compose UI、协程、DataStore），整理出以下性能可优化点。每条都已经对照真实代码校验过，未列入理论最佳实践。下面所有行号基于当前 `main` 分支。

---

## 🔴 P0：高优先级（改动小、收益大）

> **涉及的数据模型**：`ChatThreadStored`（`ChatHistoryStore.kt:23-34`）
>
> | 字段                  | 类型           | 说明                                                                |
> | --------------------- | -------------- | ------------------------------------------------------------------- |
> | `id`                  | `String`       | `"thread-${query.hashCode()}"`，同 id 覆盖（幂等去重，非多轮）      |
> | `title`               | `String`       | = `userQuery.take(60)`                                              |
> | `createdAt`           | `Long`         | 创建时间戳                                                          |
> | `preview`             | `String`       | AI 回答前 140 字                                                    |
> | `messageCount`        | `Int`          | 当前固定 2（1 问 + 1 答）                                           |
> | `messageTexts`        | `List<String>` | 早期为多轮预留，实际只用前 2 个元素                                 |
> | `userQuery`           | `String`       | 单数：用户问的那一句                                                |
> | `aiText`              | `String`       | 单数：AI 答的那一句                                                 |
> | **`restaurantsJson`** | `String`       | **★ 重字段**：`gson(List<MockMerchantDetail>)`，单条几 KB ~ 十几 KB |
>
> **存储结构**：DataStore key `history:<userId>` → value = 整个 `List<ChatThreadStored>` 的 JSON 字符串（上限 50 条）。

### 1. `ChatHistoryStore` 每次增删都全量 JSON 序列化

#### 位置

`app/src/main/java/com/wanso/app/v2/data/ChatHistoryStore.kt`

- `add()` 行 60-70
- `delete()` 行 83-92
- `pullFromServer()` 行 200-209
- `pushToServer()` 行 222-227

#### 问题

看 `add()`（行 60-70）：

```kotlin
suspend fun add(userId: String, thread: ChatThreadStored) {
    ctx.chatHistoryDataStore.edit { p ->
        val cur = gson.fromJson(p[k] ?: "[]", type)            // ① 反序列化整个 List
        val updated = listOf(thread) + cur.filter { it.id != thread.id }
        p[k] = gson.toJson(updated.take(50))                    // ② 序列化整个 List 写回
    }
}
```

`delete()` / `pullFromServer()` / `pushToServer()` 同套路：`read → Gson parse 整个 List → 改一条 → Gson toJson 整个 List → 写回 prefs`。

**写放大的成因**：只要存储模式是"一个 DataStore key 装整个 List 的 JSON"，任何写操作都得走 ① + ②。哪怕所有字段都是空字符串，加/删一条 thread 仍然要把剩下 49 条全部反序列化再重新序列化写回去。

`restaurantsJson` 是**放大器**——它是 `gson(List<MockMerchantDetail>)` 的 JSON 字符串，而 `MockMerchantDetail`（`MockData.kt:236-260`）有 19 个字段、4 个嵌套 List（`menu` / `platforms` / `sampleDishes` / `crossPlatformCompare`）。一次问答推荐 5~10 个餐厅 → 单条 `restaurantsJson` 几 KB ~ 十几 KB。50 条 × 8 KB/条 ≈ 整个 key 的 value **几百 KB ~ 1 MB**。

假设 50 条历史，单条 thread 8 KB：

| 改动                              | 写盘量             | 是否根治                                |
| --------------------------------- | ------------------ | --------------------------------------- |
| 删 `restaurantsJson` 字段（误修） | 50 × 2 KB = 100 KB | ❌ 还有写放大，只是小一点               |
| 拆索引（短期方案）                | 50 × 200 B = 10 KB | ✅ 增删动索引，`restaurantsJson` 按需读 |
| 上 Room（中期方案）               | 1 行 INSERT/DELETE | ✅ 完全没写放大                         |

**类比**：现在的写法 = 想在第 47 页加一行字 → 整本书 50 页全部重新排版打印。`restaurantsJson` 只是把每页字数变多，"重排整本书"这件事本身没变。

> `restaurantsJson` 不是写放大的"凶手"，是"扩音器"。删字段治标，拆 key / 上 Room 才治本。

#### 优化方向

按工程量从低到高：

- 短期：拆成两个 key —— 索引（id/title/preview/createdAt）单独存，`restaurantsJson` 按需读。增删只动索引。
- 中期：换 Room，每条 thread 一行，restaurants 拆子表或 JSON 列。
- 兜底（已有）：保留 `take(50)` 上限，最坏情况不会失控。

#### 风险

需要数据迁移逻辑（老数据是一次性 JSON）。中短期方案需要写一次性迁移。

---

### 2. `HomeScreen` 推荐流 LazyRow 缺 key

#### 位置

`app/src/main/java/com/wanso/app/v2/ui/screens/home/HomeScreen.kt:999`

```kotlin
items(picks) { pick -> ... }   // 没有 key
```

#### 问题

picks 是首页推荐流。一旦数据更新（下拉刷新、位置切换），整条 LazyRow 全量重组，所有 `AsyncImage` 重新触发 Coil 请求，肉眼可见的闪烁。

#### 改法

一行：

```kotlin
items(picks, key = { it.id }) { pick -> ... }
```

#### 对照

同仓库其他屏幕都已正确用 key ——

- `RecentOrdersScreen.kt:195`
- `SavedScreen.kt:506`
- `ExploreScreen.kt:849`
- `HistoryScreen.kt:177`
- `ProfileScreen.kt:1225`

只有 HomeScreen 这一处漏了。

#### 风险

无。前提是 `pick.id` 在数据流中稳定。

---

### 3. `SearchResultsScreen` 聊天流 LazyColumn 用 `items(size)` 没 key

#### 位置

`app/src/main/java/com/wanso/app/v2/ui/screens/search/SearchResultsScreen.kt:500`

```kotlin
items(messages.size) { idx -> ... }
```

#### 问题

聊天消息是流式追加的（AI 回复逐 token / 逐句 push）。每来一次更新，整个 LazyColumn 所有 item 失去身份重新重组。流式聊天场景下，滚动卡顿、输入抖动主要来自这里。

#### 改法

```kotlin
items(messages, key = { it.id }) { msg -> ... }
```

前提：message 有稳定 id。如果没有，需要在数据层补一个（生成 UUID 或用后端返回的 messageId）。

#### 风险

低。需要确认 message id 的稳定性。

---

## 🟡 P1：中优先级

### 4. `ApiClient` 拦截器每次请求都过一遍 DataStore Flow

#### 位置

`app/src/main/java/com/wanso/app/v2/data/api/ApiClient.kt:46`

```kotlin
val token = runBlocking { authStore.currentToken() }
```

#### 误判澄清

避免后续被"修"成更糟的样子：

- 这**不是**在主线程。OkHttp interceptor 跑在 dispatcher 工作线程，`runBlocking` 不卡 UI。
- 改成 `authStore.tokenFlow.first()` **没用** —— `currentToken()` 内部本来就是 `dataStore.data.first()`，两者等价。

#### 真正可优化点

- 每个请求都要：启动协程 → 订阅 DataStore Flow → `first()` → 关闭协程。一天几百个请求，这块开销能累积。
- 顺带有个潜在 race：登录后 `DataStore.edit` 落盘和下一个网络请求之间，token 可能还没刷到 in-memory cache。

#### 改法

在 `AuthStore` 内部维护一个 `MutableStateFlow<String>` 做内存缓存：

- `saveLogin()` / `clear()` 时更新它
- 拦截器里 `authStore.tokenFlow.value` 同步读
- 应用启动时一次性 load 填充

#### 风险

低。注意冷启动时缓存还没填好的窗口期，初始化要早于第一个网络请求。

---

### 5. `ChatHistoryStore.syncScope` 是无人 cancel 的顶层 scope

#### 位置

`app/src/main/java/com/wanso/app/v2/data/ChatHistoryStore.kt:40-42`

```kotlin
val syncScope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
```

#### 问题

注释写"长寿 scope，Home 离开不取消"，意图清楚。但 process 级 scope 没有任何 cancel 入口，登录/登出切换、App 退出后协程仍在跑，潜在泄漏。

#### 改法

- 把 scope 绑到 `Application` 单例（Application 提供 `appScope`）
- 或绑到 `ProcessLifecycleOwner` 的 `lifecycleScope`
- App 进程退出时统一 cancel

#### 风险

低。需注意现在依赖 `syncScope` 的调用点（搜全仓库 grep）。

---

## 🟢 P2：低优先级 / 信号不足，先不动

以下各项**已检查过没问题**或**暂无证据是瓶颈**，列入是为了避免后续重复怀疑。

| 项目                        | 结论                                                                                               |
| --------------------------- | -------------------------------------------------------------------------------------------------- |
| `maxRequestsPerHost=15`     | **不是问题**。OkHttp 默认 5，已经是 3 倍，注释也写了是给翻译预热让路。Agent 容易误判这里"过保守"。 |
| `GpsLocator` 2 分钟位置缓存 | 没看到高频调用点，暂不动。                                                                         |
| `MockData` 常驻             | 需先确认是否仍在用，再决定删除。                                                                   |
| 图片加载（Coil）            | 已配置 25% 内存 + 512MB 磁盘缓存，够用。                                                           |
| Application#onCreate        | 没有重活，启动性能 OK。                                                                            |
| Analytics                   | 已用 IO dispatcher，不阻塞 UI。                                                                    |

---

## 实施顺序建议

**Sprint 1**（零风险、可立即合）

- [ ] #2 HomeScreen LazyRow 加 key
- [ ] #3 SearchResultsScreen LazyColumn 加 key（需先确认 message id 稳定）

**Sprint 2**（动数据层）

- [ ] #4 AuthStore token 内存缓存
- [ ] #5 syncScope 生命周期绑定

**Sprint 3**（动结构、需迁移）

- [ ] #1 ChatHistoryStore 索引/内容拆分（或迁 Room）

---

## 验证方法

每条改动后建议的回归检查：

- **#2 #3**：首页下拉刷新、聊天流式回复过程中滚动 —— 卡顿/闪烁是否消失。可用 Macrobenchmark `frameDurationCpuMs` 度量。
- **#4**：抓网络包确认所有请求 Authorization 头仍正确；冷启动 + 登录后立即发请求，看 token 是否带对。
- **#5**：登录/退出反复多次，dump 协程看是否有遗留。
- **#1**：跑一轮"聊 10 个对话 → 删第 5 条 → 再聊 1 条"用例，对比改动前后 `Trace` 中 DataStore 写入耗时。

---

## 附录：关键概念澄清

> 本节汇总围绕 `ChatHistoryStore` / thread 模型 / 存储架构的几个常见疑问。不涉及改动，只澄清现状与取舍。

### A.1 会话历史为什么要写在前端

**App 必须有前端本地存储**，`ChatHistoryStore.kt:138-139` 的注释明确写了：

> 本地永远是 source-of-truth，服务端是镜像；任何网络失败本地都不受影响

原因：

- **离线可用** —— 弱网/无网场景下也能看历史
- **低延迟** —— 点开一条 thread 是本地 read（毫秒级），不是网络请求（百毫秒级）
- **省流量** —— 50 条带完整餐厅对象的历史反复来回拉，带宽成本高
- **跨设备同步是补充** —— 通过 `pullFromServer()` / `pushToServer()` 做镜像，失败不影响本地

数据分层：

| 层                | 存什么                                                   | 备注                                       |
| ----------------- | -------------------------------------------------------- | ------------------------------------------ |
| 本地（DataStore） | 整条 thread 的完整内容（含 restaurantsJson），上限 50 条 | 权威源                                     |
| 服务端            | 同样的数据                                               | 登录后同步，跨设备漫游                     |
| 匿名桶            | `history:anonymous`                                      | 登录时 `mergeAnonymousInto()` 合并到用户桶 |

### A.2 App 前端 vs Web（围绕存储这个点）

| 维度       | Web 前端                                    | Android App 前端                                                      |
| ---------- | ------------------------------------------- | --------------------------------------------------------------------- |
| 持久化手段 | localStorage（5MB 上限）、IndexedDB、Cookie | SharedPreferences / **DataStore**（本项目）/ **Room**（SQLite）/ 文件 |
| 存储容量   | 几 MB 起步，谨慎用                          | 几百 MB 不算事，本地 SQLite 表随便建                                  |
| 数据可靠性 | 用户清缓存/隐私模式就没了                   | App 卸载才丢，系统不会乱清                                            |
| 结构化查询 | IndexedDB 比较难用                          | Room 直接写 SQL，可建索引、JOIN                                       |
| 跨设备同步 | 浏览器绑定，跨设备必须靠后端                | 同样靠后端，但本地缓存能更大                                          |
| 生命周期   | 关 tab 就停                                 | 进程长期活着，要管 scope（见 perf plan #5）                           |
| 写放大风险 | 数据少，不太敏感                            | 数据多 + 全量序列化就有性能问题（见 perf plan #1）                    |

> **要点**：Web 的本地存储更"软"（随时可能被清），App 更"重"。Web 上 localStorage 全量 JSON 序列化基本没事，App 上同样的写法就是 perf plan #1 标红的问题。

### A.3 threadId 是什么

**threadId = 一次问答会话的 ID**。在当前实现下"一个 thread = 一次问答"，**不是 ChatGPT 那种长会话**。

`ChatThreadStored` 字段（`ChatHistoryStore.kt:23-34`）：

```kotlin
data class ChatThreadStored(
    val id: String,                 // threadId, 唯一标识
    val title: String,              // = userQuery.take(60)
    val createdAt: Long,
    val preview: String,            // AI 回答的前 140 字
    val messageCount: Int,
    val messageTexts: List<String>, // 当前实际只填前 2 个元素
    val userQuery: String,          // ★ 单数：用户问的那一句
    val aiText: String,             // ★ 单数：AI 答的那一句
    val restaurantsJson: String,    // AI 推荐的餐厅列表
)
```

注意 `userQuery` 和 `aiText` 都是**单数 String**，一个 thread 装的就是"1 问 + 1 答 + 推荐餐厅"。

ID 生成方式（`SearchPreload.kt:191`）：

```kotlin
id = "thread-${query.hashCode()}"
```

同一句话永远生成同一个 id —— 这是为了**幂等去重**（同一句话问两次不会产生两条历史），而不是多轮。

同步到后端时 `sessionId = t.id`（`ChatHistoryStore.kt:162`），所以 **前端的 `thread.id` = 后端的 `session.id`**，是同一个东西。

### A.4 当前是否支持同一个 thread 多轮对话？不支持。

证据：

1. **写入端**（`SearchPreload.kt:190-200`）：`messageCount = 2` 写死，`messageTexts = listOf(query, result.text)` 永远 2 条。
2. **追加逻辑**（`ChatHistoryStore.kt:67`）：`listOf(thread) + cur.filter { it.id != thread.id }` —— 同 id 覆盖，不是追加消息。
3. **读取端**（`SearchResultsScreen.kt:346-368`）：还原时只 `add` 一条 `User` + 一条 `AiRestaurants/AiText`。`messageTexts.size >= 2` 分支是老数据兜底，只取索引 `[1]`。

`messageTexts: List<String>` 这个字段是早期为多轮预留的，**但从没真正用上**，只填前两个元素。

### A.5 为什么改存储格式需要数据迁移

当前存储格式：

```
key: "history:userId"
value: 一整个 JSON 字符串 [thread1, thread2, ... thread50]   ← 全量塞一个 key
```

短期方案（新）：

```
key: "history_index:userId"        ← 只存索引（轻）
key: "thread:{id}" × 50            ← 每条 thread 各自一个 key（重，按需读）
```

中期上 Room：

```
表 chat_threads       一行一条 thread
表 chat_restaurants   thread_id 外键
```

**不迁移会怎样**：用户升级新版后，老代码写的那个大 JSON `"history:userId"` 还躺在 DataStore 里，新代码只认新格式（新 key / Room 表），**完全看不到旧数据** → 用户感受"我之前 10 条聊天记录没了"。

迁移逻辑大致是：

```
1. 读 "history:userId" → 拿到老 JSON List
2. 拆开：索引 → "history_index:userId"；每条 thread → "thread:{id}"
3. 老 key 重命名为 "history_legacy:userId"（留备份，别直接删）
4. 写 flag "migrated_v2 = true"，下次启动直接跳过
```

Room 方案有内置的 `Migration(N, N+1)` 机制，框架帮管版本号。

#### 风险

- 只能跑一次，写错 50 条历史就没了，必须做备份
- 要测各种边界（老 JSON 损坏、字段缺失、跨版本、anonymous 桶也要迁）
- 灰度发版，一旦发现 bug 已经升级的用户回不去

> **改存储格式 ≠ 改代码**，还意味着要把用户已经存在手机里的旧数据搬一遍。这步漏了就是事故级体验。

### A.6 做真正的多轮对话：要不要上 Room

**不是必须，但强烈推荐。** 三条路：

#### 方案 A：继续 JSON 全量塞（最简单，但性能爆）

`ChatThreadStored` 加个 `messages: List<MessageStored>` 字段，每来一条新消息：read JSON → parse → append → serialize → write 整个 List。

**问题**：就是 perf plan #1 标红的写放大。一问一答时是"50 条 × 餐厅 JSON"，多轮时变成"50 条 × N 条消息 × 餐厅 JSON"，**写放大随 N 线性恶化**。

#### 方案 B：DataStore 拆 key（中等，不依赖 Room）

```
history_index:userId          ← 只存 [threadId, title, preview, createdAt]
thread:{threadId}             ← 这个 thread 的完整 messages JSON
```

索引轻，列表快；追加消息只动单个 `thread:{id}` key。

**缺点**：单条 thread 的 messages 还是要 JSON parse/serialize，只是粒度从"50 条"降到"1 条"，治标不治本。

#### 方案 C：Room（做多轮的正路）

```sql
chat_threads:        一行一 thread  (id, title, created_at)
chat_messages:       一行一 message (id, thread_id FK, role, content, created_at)
                     索引: thread_id, created_at
```

- **写一条消息 = 一行 INSERT**，完全不读不写整个 thread，没有写放大
- 查一个 thread 的全部消息直接走 SQL 索引
- Room 自带 Schema 版本管理、迁移框架、事务
- **代价**：加依赖（KSP + Room runtime）、写 `@Entity` / `@Dao`、初始学习成本

#### 选型建议

| 目标                   | 选什么                                       |
| ---------------------- | -------------------------------------------- |
| 保持一问一答（像现在） | **别动**。最多按 perf plan #1 短期方案拆索引 |
| 想做 ChatGPT 式多轮    | **必须上 Room**。JSON 撑不住写放大           |
| 想做多轮但赶时间       | 先方案 B 拆 key 顶一阵，清楚这是过渡         |

> **现在一个 thread 就是"1 问 + 1 答"，没有多轮。要做真正的多轮，Room 不是可选项是必选项，否则每发一条消息都要把整段历史序列化写一遍磁盘。**
