# 进程局部缓存的跨 worker 优化方案

> 对象:`server.py` 中 4 个进程局部缓存(`_chat_response_cache`、`_restaurants_with_menu_cache`、`_rate_limits`、`_cross_menu_cache`)
> 分析日期:2026-07-05
> 配套阅读:`wanso-backend/CLAUDE.md`(「`server.py` 里的内存缓存是进程局部的」段)、`deploy/local-docker.md`
> 状态:**待评审**。本文不预设结论是 Redis,先列事实再对比方案。

---

## 一、问题陈述(再说一遍,确认是什么病)

CLAUDE.md 已经点出"17 个 gunicorn worker 意味着 17 个独立缓存视图"。本节把这个直觉量化。

### 当前 4 个进程局部缓存的清单

| 缓存 | 位置 | 数据结构 | TTL | 容量 | 跨 worker? |
|---|---|---|---|---|---|
| `_chat_response_cache` | `server.py:16005` | `OrderedDict`(LRU) | 1800s(30min) | 200 条 | ❌ |
| `_restaurants_with_menu_cache` | `server.py:5466` | 单值 dict `{ids, ts}` | 300s(5min) | 1 条 | ❌ |
| `_rate_limits` | `server.py:6109` | `defaultdict(list)` | 60s 滚动窗口 | 无上限 | ❌ |
| `_cross_menu_cache` | `server.py:16101` | `dict{(rid,other_id): (ts, pairs)}` | 7 天 | 无上限 | ❌ |

### 已经跨 worker 共享的缓存(本文不动)

| 缓存 | 存储 | 备注 |
|---|---|---|
| `menu_translations` | PG 表 | 翻译 AI 调用结果 |
| `wanso_food_term_cache` | PG 表(经 `db.py` 适配,源 SQLite `wanso.db`) | `_wanso_v9_cache_*` |

**这两类已经在 PG 里,所以不存在本文讨论的问题。** 任何新方案都不应该重复它们的轮子。

---

## 二、量化:17 worker 下的实际损失

### 损失 1:缓存命中率被 1/17 稀释

`_chat_response_cache` 的 key 是 `(message|city|lat(4位)|lng(4位)|pref_hash)`。同一用户连续两笔请求落在同一 worker 的概率是 1/17 ≈ **5.9%**。也就是说:

- **理论命中率**:同一 query 30min 内重复访问,应 100% 命中
- **实际命中率上限**(17 worker,完全随机路由):**5.9%**
- **实际收益**:省下的 Qwen 调用 ≈ 应省的 1/17

> 注:这是上限估算。如果上游负载均衡是 IP hash / sticky,实际会高些。但 gunicorn 默认 pre-fork + 反向代理 round-robin,接近随机。

### 损失 2:`_restaurants_with_menu_cache` 17 倍 SQL 重复

`get_restaurants_with_menu()`(server.py:5469)是 `/api/restaurants` 列表查询的 hot path。

- 单次成本:`SELECT DISTINCT restaurant_id FROM menu_items WHERE price >= 10000`(扫整张菜单表,粗估 50–200ms 视 PG 缓存)
- 5 分钟内 17 个 worker 各跑一次 = **17 倍无意义成本**
- 这个查询的输入**完全无差异**(没有 user-specific 维度),纯粹是浪费

### 损失 3(最严重):限流被 1/17 削弱 ⚠️

`_rate_limits` 是 `/api/chat` 的 IP 限流(CLAUDE.md 写的 "10/分钟")。17 个 worker 各持一份 dict,实际效果:

- **声称的限制**:每 IP 10/分钟
- **实际限制**:每 IP **170/分钟**(17 worker × 10)
- 攻击者从同一 IP 撞 Qwen API,每分钟能白嫖 170 次而不是 10 次

**这是安全问题,不是性能问题。** 优先级应高于其他三个缓存。

### 损失 4:`_cross_menu_cache` 7 天 TTL 在 17 worker 下基本打不中

跨平台菜单对齐(AI 调用结果)TTL 是 7 天。但单 worker 持有的 key 命中概率:

- 跨平台对 (rid, other_id) 数量级:城市数 × 每城餐厅数²,粗估数千到上万对
- 单 worker OrderedDict 无容量上限,**但 worker 重启就全清**
- gunicorn 正常 graceful reload 或容器重启时,7 天 TTL 完全失效

实际生产里这个缓存的命中率几乎为 0。R28 注释(`server.py:7567`)说"LLM align disabled — was 60s+ blocking detail",意味着**这个缓存当前只是占位**,真正修复要先把对齐改异步,本文最后一节会再提。

---

## 三、方案对比

四个备选方向,从"最不动运维"到"最一劳永逸"排序。

### 方向 A:接受现状,只修 `_rate_limits`(最小改动)

只把 `_rate_limits` 挪到 PG/Redis,其他三个进程局部缓存保持不动。

- ✅ 修掉唯一的安全问题(限流被绕过)
- ✅ 运维零新增(用现有 PG)
- ❌ 缓存命中率问题不解决
- ❌ 5min SQL 重复问题不解决

**适用场景**:如果生产环境实际不是 17 worker(待核实),或团队对引入共享存储有顾虑。

### 方向 B:全部挪到 PG(推荐 ⭐)

用项目已有的 PostgreSQL 替代进程局部 dict,新增 4 张表/键值表。`db.py` 的 SQLite→PG 适配层已就绪,代码改动以 SQL 为主。

**新增表(都 `CREATE IF NOT EXISTS`,符合 CLAUDE.md 的"无 migration 框架"约定)**:

```sql
-- 1. 限流(IP 滚动窗口)
CREATE TABLE IF NOT EXISTS rate_limit_buckets (
    ip TEXT,
    bucket_minute BIGINT,  -- epoch / 60
    hit_count INT DEFAULT 0,
    PRIMARY KEY (ip, bucket_minute)
);

-- 2. chat 响应缓存(LRU 由容量 + DELETE 老条目实现)
CREATE TABLE IF NOT EXISTS chat_response_cache (
    cache_key TEXT PRIMARY KEY,
    response_json JSONB,
    created_at BIGINT
);
CREATE INDEX IF NOT EXISTS idx_chat_cache_created
    ON chat_response_cache(created_at);

-- 3. restaurants_with_menu(单行)
-- 复用现有 SET/GET 模式,无需新表,直接用 kv 表:
CREATE TABLE IF NOT EXISTS kv_cache (
    k TEXT PRIMARY KEY,
    v JSONB,
    expires_at BIGINT
);

-- 4. cross_menu_cache
-- 同上 kv_cache,key 形如 "cross_menu:grab-xxx:sf-yyy"
```

代码层面:

- `_chat_cache_get/_put`:把 `OrderedDict` 操作换成 `SELECT ... WHERE cache_key=?` / `INSERT ... ON CONFLICT`
- `_restaurants_with_menu_cache`:换 kv_cache 一次 read,简单到几乎不需要写新函数
- `_rate_limits`:UPSERT `hit_count = hit_count + 1`,然后 `SELECT SUM(hit_count) ... WHERE bucket_minute >= (now-60s)/60`
- `_cross_menu_cache`:kv_cache,key 编码为字符串

**封装建议**:写一个 `_shared_cache.py` 模块,提供 `cache_get(key) / cache_put(key, value, ttl) / rate_limit_hit(key, max)` 三个 API。所有调用点改一次。不要再让 `server.py` 直接写 SQL(24133 行已经够长)。

- ✅ 运维零新增组件(已有 PG)
- ✅ 一并解决限流安全 + 缓存命中率 + SQL 重复
- ✅ 容器/裸金属两种部署都不需要改基础设施
- ✅ pgloader、备份、监控链路全部现成
- ❌ 比 Redis 多 ~1ms 网络/磁盘延迟(见下文「PG vs Redis 延迟」评估)
- ❌ 需要写 4 张表的清理任务(TTL 过期项不主动删,会涨)

**PG vs Redis 延迟评估**:这 4 个缓存都不是热路径(`_chat_response_cache` 命中也省了 1 次 Qwen 调用 = 数百 ms,PG 多出的 1ms 完全无感;`_rate_limits` 在请求开头,1ms 可接受;`_restaurants_with_menu_cache` 5min 一次,1ms 无意义;`_cross_menu_cache` 命中也只省 AI 调用)。**Redis 的低延迟优势在这里换不成业务收益。**

### 方向 C:Redis(标准答案但不推荐)

引入 Redis 容器 + `redis-py` 依赖。

- ✅ 性能最好(亚 ms)
- ✅ 有原生 TTL、LRU、限流原语(`INCR` + `EXPIRE`)
- ✅ 业界标准,文档丰富
- ❌ **运维新增组件** —— 需要在 VPS 上 systemd 起一个 redis-server,或 docker-compose 加 redis 容器
- ❌ 备份策略要重新设计(`BGSAVE` / AOF)
- ❌ 监控要加(grafana redis dashboard)
- ❌ Docker 本地开发要多起一个容器(`deploy/local-docker.md` 当前是单容器 + SSH 隧道,加 Redis 复杂度上升)
- ❌ 团队当前没有 Redis 运维经验(基于 CLAUDE.md 风格推断)

**判断标准**:如果 PG 方案在压测下证明延迟不够(目前判断不会),再上 Redis。否则引入新运维负担换不到业务收益。

### 方向 D:文件 + flock(不推荐,列出供排除)

用 `~/.cache/wanso/*.json` + `fcntl.flock` 共享状态。

- ✅ 无组件依赖
- ❌ Windows 不能用 flock(项目还在 Windows 开发,见 user.md memory)
- ❌ 并发写性能极差
- ❌ 没有原生的 TTL/LRU
- ❌ **已被 CLAUDE.md 父级规则否定**("裸机 systemd 无容器"已被推翻,但文件锁方案从来不是好选择)

**结论**:不采用。

---

## 四、性能收益评估:Redis 真正能解决什么?

> 这一节直接回答两个问题:**引入 Redis 主要解决哪些痛点?能优化性能吗?**
> 短答:核心痛点它能解决(但 PG 也能),性能优化它帮不了多少。

### 4.1 Redis 能解决的痛点(和 PG 共有)

下表 4 个缓存的"跨 worker 共享"问题,Redis 和 PG **都能解决**,选哪个不改变这部分收益:

| 痛点 | 不修的代价 | Redis 解决? | PG 解决? |
|---|---|---|---|
| `_rate_limits` 17 倍削弱 | 实际限流 170/分(声称 10/分) | ✅ | ✅ |
| `_chat_response_cache` 命中率上限 1/17 ≈ 5.9% | 缓存基本白搭 | ✅ | ✅ |
| `_restaurants_with_menu_cache` 17 倍重复 SQL | 5min 内重复跑大查询 | ✅ | ✅ |
| `_cross_menu_cache` 重启即清 | 当前无数据(R28 已禁用) | ✅(但无意义) | ✅(同上) |

**这部分收益与选 Redis 还是 PG 无关**。如果只看"修不修",答案是"修";"用 Redis 还是 PG"是另一个问题(见 4.2、4.3)。

### 4.2 Redis 比 PG 多出的"性能优势"——本项目基本用不上

Redis 的核心优势是**亚毫秒延迟 + 原生 TTL/LRU/原子计数器**。但放到本项目里:

| 缓存 | 一次命中省下的成本 | Redis vs PG 单次差异 | 实际意义 |
|---|---|---|---|
| `_chat_response_cache` | 1 次 Qwen 调用 ≈ 数百 ms ~ 数秒 | ~0.2ms vs ~1ms | <0.1% 量级,**无感** |
| `_restaurants_with_menu_cache` | 1 次大 SQL ≈ 50–200ms | ~0.2ms vs ~1ms | <1% 量级,无感 |
| `_rate_limits` | 不省下游,只是判断 | ~0.2ms vs ~1ms | 请求开头,无感 |
| `_cross_menu_cache` | 当前数据为空(R28) | — | 不存在 |

**结论**:这些缓存的命中收益都在"省 LLM / 省大 SQL"层面(数百 ms 量级),Redis 比 PG 快的 0.8ms 在这个尺度下测不出来。用 PG 替代 Redis 不会引入用户可感知的延迟。

> 📌 **2026-07-05 修订**:`_chat_response_cache` 已采纳分层缓存方案(§9),只缓存 intent + 候选 SQL,**不缓存最终响应**。这把单次命中节省从"2–4 秒"降到"0.5–1 秒"(因为 QWEN_MODEL 每次仍重打)。Redis 的延迟优势在更小的总收益面前,更不值得引入。

### 4.3 Redis 救不了的性能瓶颈(真正的性能问题在别处)

如果诉求是"让 `/api/chat` 更快",**Redis 帮不上**,真正的瓶颈在别处:

1. **2 次 LLM 调用**(intent 分类 `QWEN_FAST_MODEL` + 响应生成 `QWEN_MODEL`)— 数秒级,占请求总耗时 80%+
2. **server.py 24,133 行单文件**,17 worker 启动各 import 一次(死代码也会被解析)
3. **GPS bbox + 菜单 LIKE 的候选 SQL**,~50–200ms
4. **17 个重复的 `def _wanso_finalize_chat_response_for_app`**(死代码)虽不直接影响运行时,但增加 import 时间

**性能优化优先级排序**(收益从大到小):

1. todo.md 那条"intent 调用是否传了全部 history"—— 减 token 直接降 LLM 延迟
2. fast path 修复(详见 `analysis/fast-path-design-issue.md`)—— 该走 AI 的别浪费 1 次调用
3. server.py 瘦身(删 17 个重复 def + 1.5–2 万行死代码)
4. **缓存统一(Redis 或 PG)**— 收益最小

**一句话**:纯优化性能 → 别上 Redis,先做前三项,收益比缓存大一个数量级。

### 4.4 Redis 在本项目真正有价值的场景(未来可能)

下面这些场景 Redis 才有 PG 替代不了的优势,目前本项目都还没到那一步:

| 场景 | Redis 优势 | 当前本项目状态 |
|---|---|---|
| **会话状态 / chat history**(每条消息读写) | 真正热路径,Redis 比 PG 快一个量级 | chat history 还在 PG(单次写、多次读,频次低) |
| **短 TTL 缓存**(秒级失效) | 原生 `EXPIRE`,PG 表清理成本高 | 当前最短 TTL 是 60s(限流),PG 完全够 |
| **跨 worker 缓存失效广播**(Pub/Sub) | 原生 Pub/Sub | 暂无需求 |
| **限流原子计数** | `INCR`+`EXPIRE` 一行搞定 | PG UPSERT+SELECT 也能做,稍繁琐 |

**判断标准**:出现"每请求多次读写"或"秒级失效"或"跨 worker 实时通知"需求时,再上 Redis。否则引入新运维负担换不到业务收益。

### 4.5 给决策画一张表

| 你的目标 | 推荐路径 |
|---|---|
| 修 `_rate_limits` 安全问题 | PG(`rate_limit_buckets` 表),无需 Redis |
| 让 `/api/chat` 更快 | **不修缓存**,先做 todo.md + fast path + server.py 瘦身 |
| 减少生产 PG 负载 | PG 方案(L1/L2 二级缓存),无需 Redis |
| 团队想借机引入标准缓存中间件 | Redis,但要明确"为未来场景付运维成本" |
| 不确定 | 先 PG 方案做 4 阶段,后续如果出现 4.4 表里的需求,再迁 Redis |

---

## 五、推荐路径

**方向 B(PG)+ 分阶段**。

### 阶段 1:紧急 —— 修 `_rate_limits`(1–2 小时)

仅这一项有安全影响,优先单独立项 PR。

1. 新增 `rate_limit_buckets` 表(在 `init_analytics_db` 末尾 `CREATE IF NOT EXISTS`)
2. 改 `check_rate_limit()`(server.py:6110)走 PG UPSERT + SUM
3. 加 `bucket_minute` 索引(过期清理用)
4. 加一个轻量清理任务:每次写入时 1% 概率 `DELETE WHERE bucket_minute < now-3600`(不引入 cron)

**验收**:
- 单 IP 连发 11 次 `/api/chat`,第 11 次必返 429(无论落到哪个 worker)
- 生产 17 worker 部署后,`SELECT COUNT(*) FROM rate_limit_buckets WHERE ip='某测试IP'` 5 分钟内应 ≤ 5 条(60s 窗口)

### 阶段 2:`_restaurants_with_menu_cache` 挪到 PG(0.5 天)

1. 在 `init_analytics_db` 加 `kv_cache` 表
2. `get_restaurants_with_menu`(server.py:5469)改读 `kv_cache`
3. 保留进程局部二级缓存(L1 PG,L2 dict),L2 仍 5min TTL —— 拿到 L1 之后还能再省一次 PG 往返,几乎免费

**验收**:
- 重启 worker 后 L1 命中(L2 冷启动时不重新跑 SQL)
- PG `SELECT * FROM kv_cache WHERE k='restaurants_with_menu'` 应有 1 行

### 阶段 3:`_chat_response_cache` 挪到 PG(1 天)

> ⚠️ **本阶段已被 §9 修订** —— 整响应缓存改为分层中间结果缓存(只缓存 intent + 候选,不缓存 AI 文案)。详见 §9.7。

1. ~~`chat_response_cache` 表 + 索引~~ → 改为 `chat_pipeline_cache` 表(jsonb 存 intent + candidates)
2. ~~`_chat_cache_get/_put` 改 SQL~~ → 改为 `_chat_pipeline_cache_get/_put`
3. 加清理任务:每次写入时 5% 概率 `DELETE WHERE created_at < now-3600`(1 小时,大于 30min TTL)
4. ~~保留 `_wanso_safe_chat_cache_put` 包装层~~ → **删除**所有调用点(§8 索引列了 12 处)

**验收**(修订):
- 同一 query 第二次访问:无 QWEN_FAST 调用日志,无候选 SQL 日志
- 同一 query 第二次访问:**仍有** QWEN_MODEL 调用(响应文案每次重新生成)
- 缓存表行数应稳定在 ≤ 200(由容量上限清理保证)

### 阶段 4(可选):`_cross_menu_cache` 挪到 PG

只在 R28 注释提到的那条「LLM align 改异步 prewarm」修复后再做。当前 `ai_pairs = []`(server.py:7570)意味着这缓存实际不打数据,挪到 PG 没意义。

---

## 六、风险与已知坑

### 风险 1:`db.py` 的 `?` → `%s` 重写跳过字符串字面量

`db.py:23` 的重写规则:别写 SQL 字符串里包含字面 `?` 的代码。本方案的所有 SQL 都是模板化参数,没有这个风险。

### 风险 2:17 worker 同时打 PG,连接池

每次缓存命中多一次 PG round-trip,17 worker × QPS 会让连接池吃紧。缓解:

- `_chat_response_cache` 命中后,先把 json 解出来塞回 `_chat_response_cache`(L2)再返回 —— 同 worker 二次命中直接走 L2
- 给 `_shared_cache.py` 的连接复用 `db.py` 的连接池(`db.connect()` 已 pooled)
- 监控 PG `pg_stat_activity` 的连接数,如果新增 17+ 个常驻连接,考虑 `pgbouncer`

### 风险 3:cache stampede(雪崩)

30 分钟 TTL 到期时,17 worker 同时 miss → 同时打 Qwen。当前 OrderedDict 实现也存在(只是 17 倍概率分散)。缓解:

- 给 `_chat_response_cache` 加 per-key lock(key 级 mutex)
- 或者接受(成本就是 1 次 Qwen 调用,影响小)

### 风险 4:`_rate_limits` 改成 PG 后,PG 不可用时限流也失效

PG 短暂故障时,缓存读失败 → 当前实现会"放行"(默认 fail-open)。安全考虑可选 fail-closed(故障时一律拒绝),但用户体验差。**建议 fail-open + 报警**(给 `analytics.py` 的 crash 表写一条)。

### 风险 5:CLAUDE.md 行号失真

文档里写的 `server.py:2649–4926`(chat 流水线)在 24133 行的当前文件里全部失效。本文使用的行号是 2026-07-05 实测,如果接下来做 server.py 瘦身(删 17 个重复 def)会再次全盘失真。建议:改 server.py 大幅瘦身后,本文行号统一以 grep 重新核对。

---

## 七、和 CLAUDE.md 的对应关系

CLAUDE.md 「Backend 特有的编辑规则」中相关条目:

- "**`server.py` 里的内存缓存是进程局部的**" → 本文是该规则的展开。修改后该规则需补充一句"L1 PG / L2 进程内 dict 二级缓存"。
- "**inline schema 变更去和其他 schema 一起**" → 阶段 1 的 `rate_limit_buckets` 表加在 `init_analytics_db` 末尾,不是单独建文件。
- "**幂等(`IF NOT EXISTS`)**" → 本文所有 `CREATE TABLE` 都遵守。
- "**严格食物词检测器**...有 exclude/weak/strong 同义词列表" → 与本文无关,本文不动 chat 流水线内部。

---

## 八、代码索引(2026-07-05 实测行号)

| 关注点 | 位置 |
|---|---|
| `_chat_response_cache` 定义 | `server.py:16005` |
| `_chat_cache_lock` / `_CacheLock` | `server.py:16004` / `16006` |
| `_CHAT_CACHE_TTL` / `_CHAT_CACHE_MAX` | `server.py:16007` / `16008` |
| `_chat_cache_key`(key 编码) | `server.py:16160` |
| `_chat_cache_get` / `_chat_cache_put` | `server.py:16177` / `16188` |
| `_wanso_safe_chat_cache_put`(调用层封装) | `server.py:12099` + 多个重定义 |
| `_restaurants_with_menu_cache` | `server.py:5466` |
| `get_restaurants_with_menu` | `server.py:5469` |
| `_RESTAURANTS_WITH_MENU_TTL` | `server.py:5467` |
| `_rate_limits` / `check_rate_limit` | `server.py:6109` / `6110` |
| `_cross_menu_cache` / `_CROSS_MENU_TTL` | `server.py:16101` / `16102` |
| `_wanso_v9_cache_connect`(SQLite,已跨 worker) | `server.py:4855` |
| `menu_translations` 表(PG,已跨 worker) | `server.py:150` + `ensure_translation_cache_table` |
| intent 调用点(QWEN_FAST_MODEL) | `server.py:13796-13801` |
| intent 结果解析 | `server.py:13806` |
| `_nearby_by_food`(候选 SQL) | `server.py:2668-2735` |
| `_nearby_top_rated`(候选 SQL fallback) | `server.py:2738-2766` |
| `_wanso_safe_chat_cache_put` 调用点(8 处) | `server.py:13522 / 13659 / 13854 / 13956 / 14020 / 14114 / 14228 / 14238 / 14259 / 14852 / 15829 / 15876` |

---

## 九、修订:分层缓存策略(2026-07-05 已采纳)

> 前文 §3–§5 默认把 `_chat_response_cache` 当作"整响应缓存"来评估。**实际工程上不应该缓存整个响应** —— 把 AI 生成的 greeting/followup 也缓存会抹平模型 `temperature > 0` 的多样性,违反 toC 列表推荐产品的体验预期。
>
> **采纳方案**:把缓存边界从"整响应"收缩到「候选 SQL 完成后、AI 响应生成前」为止。本节内容**覆盖** §4 部分估算和 §5 阶段 3 的描述。

### 9.1 为什么不缓存最终响应(产品权衡)

`/api/chat` 响应是 `{greeting, picks, followup}` 三段结构。两类内容缓存策略**应该完全不同**:

| 内容 | 来源 | 同 query 重复时用户期望 | 该缓存? |
|---|---|---|---|
| `picks`(推荐哪几家) | SQL 候选 + 距离/评分排序 | **一致**(回看时想看到同样的店) | ✅ |
| `greeting` / `followup`(文案) | LLM 生成(temperature > 0) | **有变化**(每次打开 app 想看不同措辞) | ❌ |

业界共识:**列表/事实型 → 缓存;生成型/创意型 → 不缓存**(美团、Yelp 都这么做)。当前 `_chat_cache_put`(server.py:16188)缓存整个响应,把 greeting 也定型,**先到先得**(第一个用户决定后续所有人看到的文案)。

### 9.2 已采纳方案:缓存到候选 SQL 为止

**缓存边界**:在 chat pipeline 里切到「候选 SQL 完成后、AI 响应生成前」。

```
请求入口
  ↓
[1] 限流
  ↓
[2] 中间结果缓存命中? ─── 是 ──┐
  ↓ 否                       │
[3] 语言检测 + 会话状态恢复    │
  ↓                          │
[4] Local intent fastpath     │
  ↓                          │
[5] AI intent 分类 ──────────┤  ← 缓存这段输出(intent dict)
  ↓                          │
[6] 候选 SQL(_nearby_by_food 等) ← 缓存这段输出(candidates)
  ↓                          │
  ←──────────────────────────┘
[7] Fast path 命中? ─── 是 ──→ 模板返回(fast path 基于规则,本身不参与缓存)
  ↓ 否
[8] AI 响应生成(QWEN_MODEL) ── 每次重新调用,生成不同文案
  ↓
返回(不写整响应缓存)
```

**缓存内容**(2 项):
- `intent`(server.py:13806):QWEN_FAST_MODEL 输出的 dict,含 `intent`、`food_keywords`、`required_elements`、`detected_language` 等
- `candidates`(server.py:2668 / 2738 附近):候选餐厅列表 + 附带的菜单/距离信息

**不缓存**(1 项):
- 最终响应(QWEN_MODEL 输出的 greeting / picks reason / followup)

### 9.3 收益估算调整(覆盖 §4 旧估算)

| 指标 | 整响应缓存(旧) | 分层缓存(新) |
|---|---|---|
| 省 intent LLM 调用(QWEN_FAST) | ✅ | ✅ |
| 省候选 SQL | ✅ | ✅ |
| 省 QWEN_MODEL 响应生成 | ✅ | ❌(每次重打) |
| 文案多样性 | ❌(全用户同文案) | ✅(每次重新生成) |
| 单次节省延迟 | ~2–4 秒 | ~0.5–1 秒 |
| 单次节省成本 | 2 次 LLM 调用 | 1 次 LLM 调用(intent) |

**收益减半**(只省 intent + SQL,不省 QWEN_MODEL),但**符合 toC 体验**。

**对存储选型的影响**:收益减半后,Redis 相比 PG 的延迟优势(0.8ms)更不重要。**§4 结论"PG 而非 Redis"在分层缓存下更站得住。**

### 9.4 实现方案

#### 选项 A:单一中间结果缓存(推荐 ⭐)

把 intent + candidates 合并成一个 dict 存:

```python
_chat_pipeline_cache = OrderedDict()  # 替代 _chat_response_cache
_PIPELINE_CACHE_TTL = 1800            # 30min(同当前)
_PIPELINE_CACHE_MAX = 200

# value 结构
{
    "intent": {"intent": "search", "food_keywords": [...], ...},
    "candidates": [{"id": "grab-5-xxx", "name": "...", ...}, ...],
    "timestamp": 1751234567.89,
}
```

读写时机:
- **写**:候选 SQL 完成、Fast path 判定**之前**(server.py:2668 之后)
- **读**:限流通过后、语言检测之前

调用点改动:
- 新增 `_chat_pipeline_cache_get(key)` / `_chat_pipeline_cache_put(key, intent, candidates)`
- 删除 `_wanso_safe_chat_cache_put` 的所有调用(§8 索引列了 12 处)
- 删除 `_chat_response_cache`(server.py:16005)

#### 选项 B:两层独立缓存

- `_intent_cache`:key 同当前,value 是 intent dict
- `_candidates_cache`:key 是 intent 输出的 hash,value 是 candidate list

**取舍**:选项 B 更"正交",但工程上要写两套 get/put + 两次 round-trip。**选项 A 简单(一次写、一次读),TTL/容量/LRU 全套只配一份**。

**推荐选项 A**。

### 9.5 Fast path 与分层缓存的关系

Fast path(server.py:6410+ 的 `_strict_food_chat_response`)在候选之后、AI 响应生成之前触发。在分层缓存下:

- 缓存命中(intent + candidates 已就位)→ fast path 仍执行(基于规则判定)
- Fast path 命中 → 模板返回(模板对同一 query 一致,**用户能接受**,因为 fast path 本来就是规则化的明确意图场景)
- Fast path 不命中 → 走 QWEN_MODEL(每次重新生成,文案有变化)

**关键**:分层缓存不影响 fast path 的行为,只影响"是否重复跑 intent + SQL"。

### 9.6 命中维度优化(独立但建议捆绑)

无论选整响应缓存还是分层缓存,命中 key 的归一化问题依然存在。建议把本节方案与"normalize message + lat/lng 3 位精度"两条优化**捆绑实施**:

- normalize 提升命中率(避免 "phở" / "pho" 互相 miss)
- lat/lng 3 位精度(110m)更符合"附近推荐"语义
- 与分层缓存叠加,综合命中率从 5.9% 拉到 15–25%

(命中优化的详细方案见独立文档,本文不展开。)

### 9.7 阶段计划(覆盖 §5 阶段 3)

原 §5 阶段 3 是"挪整响应缓存到 PG"。**采纳分层缓存后,改为**:

#### 阶段 3(修订):分层中间结果缓存

1. 删除 `_chat_response_cache` + `_wanso_safe_chat_cache_put` 的所有调用点
2. 新增 `_chat_pipeline_cache`(OrderedDict,30min TTL,200 容量)
3. 在 intent 调用前后(server.py:13796)和候选 SQL 之后(server.py:2668 之后)插入读写
4. 暂不挪 PG:**先在进程局部验证逻辑正确,再考虑跨 worker 共享**
5. 验证通过后,再按 §5 阶段 1/2 的方式挪到 PG(jsonb 字段)

**验收**:
- 同一 query 第二次访问:logs 显示无 QWEN_FAST 调用,无候选 SQL 日志(`_nearby_by_food: term=...`)
- 同一 query 第二次访问:logs 显示**有** QWEN_MODEL 调用(响应仍重新生成)
- 响应 greeting 文案每次不同(随机抽查 5 次同 query,greeting 至少 3 次不同)
- 缓存容量稳定 ≤ 200

### 9.8 风险(补充 §6)

#### 风险 6:候选 list 序列化大小

每个候选可能 100–500 字节(含菜单片段),5–15 个候选 = 1–7 KB。OrderedDict 200 容量 → 上限 ~1.4 MB,**单 worker 内存可接受**。

挪到 PG 后,jsonb 字段单行 ~7 KB,200 行 = ~1.4 MB,PG 完全无压力。

#### 风险 7:候选结果过时

候选包含菜单/价格,refresh worker 更新餐厅数据后,候选可能短暂过时(直到 30min TTL 到期)。

缓解:
- 当前 TTL 30min 已合理(refresh 周期通常更长)
- 如果发现命中数据明显过时,可加 cache invalidation hook:refresh 完成后扫 `chat_pipeline_cache` 删除受影响 key(但要从候选 list 里反查餐厅 id,需要 jsonb 查询)
- 兜底:`/api/restaurant/{id}` 详情接口每次实时查 DB,详情永远是最新数据;列表层短暂的过时用户基本无感
