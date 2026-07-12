# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 仓库结构

这是 **Wanso**(越南餐厅聚合 App,整合 GrabFood + ShopeeFood)的 monorepo。三个互相独立的组件并列,没有共享构建系统。

- `wanso-backend/` — Python FastAPI 后端(大部分工作的焦点)
- `my-app-android/` — Kotlin Android 客户端(Gradle KTS)
- `my-app-ios/` — Swift iOS 客户端(Xcode 工程)
- `.github/modernize/java-upgrade/` — agent 工具调用 hook 日志(不是源码)

## 后端:运行时与部署实情

**后端只在生产 VPS 上运行。** 本地没有开发入口 —— 所有脚本都假设以下路径。

- 主机:`gbz20@49.213.71.8:234`(SSH 端口 234)
- 应用目录:`/home/stackops/wanso-backend/`
- Python:应用目录内的 `venv/`(Linux 环境,Windows 上无法使用)
- 日志:`/var/log/wanso/{access,error,stdout,stderr}.log`、`/var/log/wanso/refresh_worker*.log`
- 两个 systemd 单元:
  - `wanso.service` — gunicorn + 17 个 UvicornWorker 进程,绑定 `127.0.0.1:8000`(前面有 nginx/caddy 反代)
  - `wanso-refresh.service` — 运行 `wanso_realtime_refresh_worker.py`,后台刷新餐厅数据
- 部署形态:**裸机 systemd,无容器**。详见 `wanso-backend/docs/deployment-analysis.md`。

**VPS 侧常用命令:**
```bash
sudo systemctl restart wanso                # 应用代码改动
sudo journalctl -u wanso -f                 # 应用日志(实时)
sudo journalctl -u wanso-refresh -f         # 刷新 worker 日志
tail -f /var/log/wanso/error.log
sudo systemctl status wanso --no-pager -l
```

**从远程同步代码到本地**(需要读取/运行只在 VPS 上的代码时):
```bash
bash wanso-backend/scripts/remote-pack.sh   # SSH 到远程打包,产出 /tmp/wanso-backend.tar.gz
bash wanso-backend/scripts/local-fetch.sh   # scp 下载,带 -k 解压(保留本地已有文件)
```
说明:`local-fetch.sh` 使用 `tar -xzkf`,**不会覆盖**本地新增的文件。远程 tar 排除 `venv/`、`*.bak*`、`*.db*`、`grab_tokens.json`、`img_cache/`、`avatars/`、调试目录 —— 修改前请先看 `remote-pack.sh` 中的排除清单(那是事实标准)。

## 关键陷阱(改后端代码前必读)

### 1. `db.py` 是 SQLite → PostgreSQL 兼容层

代码看起来像在用 `sqlite3`(用 `?` 占位符、`sqlite3.Row`、`conn.row_factory = ...`、`conn.create_function(...)`),**但真实数据库是 PostgreSQL**。`db.py` 在运行时把 `?` 改写成 `%s`、把 `datetime('now')` 翻译成 PG 表达式,**并静默吞掉 DDL 错误**(因为表结构最初是 `pgloader` 建的)。

影响:
- 不要假设 SQLite 语义(没有 `INSERT OR REPLACE`、没有 `rowid` 等)。
- DDL "报错"通常只是表/列已存在 —— 静默 no-op 是预期的,**不要修复**。
- 新增字段要写幂等的 `ALTER TABLE ... ADD COLUMN IF NOT EXISTS ...`,并在模块导入或启动时执行(`server.py` 顶部就是这么做的)。

### 2. 硬编码绝对路径无处不在

几乎所有模块都硬编码了 `/home/stackops/wanso-backend/...`。`.env` 就是从这个绝对路径加载的(`db.py`、`auth.py`)。改代码时**沿用这个约定** —— 除非用户明确要求支持本地运行,否则不要引入相对路径。

### 3. `server.py` 是 283 KB 的单体文件

绝大多数 HTTP handler 直接定义在 `server.py` 里(搜索、餐厅详情、聊天、菜单翻译、崩溃上报、AI 标签、explore、insights、地点解析等)。其他模块的 router 在 app 构造时挂载:

```python
# server.py 第 821 行附近
app = FastAPI(title="Wanso API", docs_url="/docs")
app.include_router(img_proxy.router)         # img_proxy.py —— /img?url= CDN 代理,带磁盘缓存
app.include_router(auth_router)              # auth.py —— JWT 认证、OAuth、邮箱验证
app.include_router(analytics_router)         # analytics.py —— 用户行为埋点
# 文件末尾:wanso_app_refresh_router、wanso_refresh_router
```

`ai_endpoints.py` **不**导出 router —— 它导出 `register_ai_endpoints(app, qwen_client, ...)` 然后被程序化挂载。新增 AI 接口时,跟随周围代码已有的风格。

### 4. AI 集成:Qwen via DashScope(OpenAI 兼容客户端)

所有 AI 调用都走 `server.py` 里那一个 `qwen_client = OpenAI(base_url=QWEN_BASE_URL, ...)`。模型(来自 `.env`):
- `QWEN_MODEL` —— 聊天 / followup
- `QWEN_FAST_MODEL` —— 意图识别
- `QWEN_TRANSLATE_MODEL` —— 菜单翻译(缓存在 `menu_translations` 表)
- `QWEN_SUMMARY_MODEL` —— 餐厅深度总结

菜单翻译是**批量调用、带缓存、走后台任务**的(`_bg_translate_worker`)。务必复用缓存,不要逐条调用模型。

### 5. 多语言与去重音符号

- 目标语言:`vi`、`zh`、`en`(翻译还支持 `ja`、`ko`)。
- 越南语:`remove_diacritics()` + 每个连接注册一次 `unaccent()` SQL 函数(在 `get_db()` 里)。
- 中文:OpenCC `t2s` 作为**兜底**,把 AI 偶尔输出的繁体强制转简(`_to_simplified_chinese`)。
- `detect_lang()` 是个轻量启发式 —— 不调 AI。

### 6. 敏感文件

`.env`(数据库密码、QWEN_API_KEY、JWT_SECRET、SMTP、GOOGLE_MAPS_API_KEY、RESEND_API_KEY)、`grab_tokens.json`、`wanso.db*`、`avatars/` 都是真实生产数据。**绝对不要提交到 git。** `scripts/remote-pack.sh` 的排除清单就是"什么不能离开 VPS"的事实标准。

## 表结构与迁移

没有迁移框架。schema 是这样建立的:
1. `pgloader` 初次导入(历史遗留,从 SQLite 迁过来 —— VPS 上的 `.pre_pg_backup/` 是迁移前快照)。
2. 模块导入时执行幂等的 `ALTER TABLE ... ADD COLUMN IF NOT EXISTS`(`server.py` 顶部、`init_auth_db`、`init_analytics_db`、`register_ai_endpoints` 内)。

新增字段:在同样的位置加一条幂等 `ALTER TABLE`,然后 `sudo systemctl restart wanso`。

## 后台刷新 worker

`wanso_realtime_refresh_worker.py` 是一个**独立进程**(`server.py` 不 import 它)。它:
- 读 `refresh_config.json` 获取调参(延迟、批量、stale 阈值、headless 开关)。
- 把 `restaurants.last_updated` 当作工作队列。
- **Grab**:需要 `grab_tokens.json`(由 `grab_token_refresher.py` 刷新)或 `GRAB_PASSENGER_TOKEN` / `GRAB_PASSENGER_JTI` 环境变量。
- **Shopee**:用 **Playwright + 响应拦截**。需要在 `sf_browser_data/` 有一个登录过的浏览器 profile。如果 Shopee 改了响应结构或 VPS 浏览器掉登录,先看 `/var/log/wanso/refresh_worker.log`。
- 触发/查询刷新的 admin 接口在 `wanso_refresh_admin.py`(在 `server.py` 末尾挂载),设了 `ADMIN_REFRESH_SECRET` 时需要带 `X-Admin-Secret` 请求头。

## 编辑代码时需要保留的约定

- **匹配所在文件的现有风格** —— 代码库中英文注释混用很常见,跟随周围的语言即可。
- **不要把 `?` 改成 `%s` 或反之** —— `db.py` 会自动翻译,两种写法都能跑。
- **不要引入测试框架、`requirements.txt`、`pyproject.toml`** —— 除非用户明确要求。依赖是直接装进 VPS 的 venv 的(`venv/bin/pip install ...`)。加这些文件会暗示一种和实际部署方式不符的工作流。
- **不要重命名 `server.py`** —— systemd 单元硬编码了 gunicorn 入口 `server:app`。
- **不要加 `.bak` 清理逻辑** —— `*.bak*` 是有意从 tar 包排除的,用户自己手动管理。

## 仓库里的临时性脚本(非运行服务的一部分)

- `wanso-backend/patch_*.py` —— VPS 侧的一次性修复脚本(place、geocode、smart-fallback)。`_v2`–`_v7` 是迭代版本,不是抽象。VPS 上跑一次就完事,之后忽略。
- `wanso-backend/debug_*.py`、`probe_*.py`、`test_shopee_*.py` —— 针对线上 Shopee API 的调研脚本。**不属于运行中的服务**。当作草稿看待。
- `wanso-backend/import_data.py` —— 批量导入餐厅/菜单。VPS 上手动运行。
- `wanso-backend/backup.sh` —— DB 备份小工具。
