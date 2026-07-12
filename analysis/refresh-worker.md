# Refresh Worker 的作用

> 整理自 2026-06-22 关于 `wanso_realtime_refresh_worker.py` 的对话。
> 详细架构见 `CLAUDE.md` 的 "Refresh worker architecture" 一节。

## 核心作用:**自动刷新餐厅数据**

### 为什么需要它

Grab / ShopeeFood 上的餐厅数据**会变** —— 菜单更新、价格调整、上下架、餐厅倒闭。如果不定期刷新,数据库里的信息会越来越陈旧,推荐出来的餐厅可能是已经关门的、价格是上个月的。

worker 就是解决"数据新鲜度"的后台进程。

### 它干什么(数据流)

```
Grab API / Shopee API
       │
       ▼
wanso_realtime_refresh_worker.py  (轮询)
       │
       ▼
wanso.db (SQLite,真实 sqlite3) ← worker 的 scratch/queue
       │
       ▼ mirror_restaurant_to_postgres()
       ▼
PostgreSQL (app-facing DB) ← FastAPI 读这里
```

关键:**worker 不直接写 PG**,而是先写本地 SQLite,再 mirror 到 PG。这样 worker 崩了不会污染 app-facing DB。

### 两种刷新触发

1. **自动轮询**:每隔 N 小时检查 `restaurants.last_updated`,把超过阈值的(`grab=24h`、`shopee=72h`)加进队列
2. **手动触发**:admin 端点 `POST /admin/refresh/restaurant/{id}` / `/admin/refresh/city/hcmc`(靠 `X-Admin-Secret` 鉴权)

### 两个平台 refresher(`wanso_realtime_refresh_worker.py`)

| Refresher | 数据源 | 依赖 |
|---|---|---|
| `GrabRefresher`(worker:228) | Grab 乘客端 portal API | `grab_tokens.json`(由 `grab_token_refresher.py` 用 Playwright 定期刷)或 `GRAB_PASSENGER_TOKEN` / `GRAB_PASSENGER_JTI` 环境变量 |
| `ShopeeRefresher`(worker:387) | **Playwright + 响应拦截**(打开浏览器登录态,嗅探 ShopeeFood 的 API 响应) | `sf_browser_data/` 登录态目录;VPS 浏览器掉登录就废 |

### 为什么本地不需要它

- **本地目的是看接口 + 调代码** —— `/api/restaurants`、`/api/chat` 这些读 PG 里的数据就够,不需要再往里灌新数据
- worker 起来后没 token / 没登录态会一直报错刷屏,干扰你看日志
- Playwright + chromium 在容器里装要拉一堆系统库,镜像变 1.5GB+

**所以本地方案里,worker 是被明确排除的。** 生产上它跑在 VPS 的 `wanso-refresh.service` 里,跟本地没关系。

### 什么时候本地需要起 worker

- 你要调试 worker 本身的代码
- 你要测某个具体餐厅刷新后的 `/api/restaurant/{id}` 返回值
- 你要复现一个数据刷新相关的 bug

这些场景出现时再加 worker 容器(`docs/local-docker.md` 里的"不在本次范围"一节就是为这个留的口子)。

### 相关文件

- `wanso_realtime_refresh_worker.py` — worker 主程序(独立进程,**不被 server.py import**)
- `wanso_refresh_admin.py` — admin 触发端点(`/admin/refresh/*`)
- `wanso_app_refresh.py` — 客户端可见的刷新端点(`/api/app/*`)
- `refresh_config.json` — 运行时配置(批次大小、延迟、headless 等)
- `grab_token_refresher.py` — Playwright 刷 Grab token(被 worker 通过 `os.system()` 调)
- `install_realtime_refresh.sh` — 首次安装脚本,装 systemd unit + 依赖 + patch admin 路由
