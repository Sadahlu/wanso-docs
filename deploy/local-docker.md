# 本地 Docker 跑 wanso-backend

## 前置

- Docker Desktop for Windows 已启动
- 能 SSH 到 VPS:`ssh -p 234 gbz20@49.213.71.8`

## 链路全景

容器内代码连 PG 走的是「容器 → 宿主机 → SSH 隧道 → 生产 PG」四跳,任何一断整条链就废:

```
┌─────────────────────────────────────────────────────────────────┐
│ Docker 容器 wanso-backend                                         │
│   PG_HOST=host.docker.internal:5432                              │
│   ↑ host.docker.internal 由 docker-compose.yml 的 extra_hosts    │
│     「host.docker.internal:host-gateway」解析到宿主机             │
└──────────────────┬──────────────────────────────────────────────┘
                   │ host-gateway 路由
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│ Windows 宿主机                                                    │
│   127.0.0.1:5432  ← ssh.exe 在监听(SSH 客户端 -L 转发的本地端口) │
└──────────────────┬──────────────────────────────────────────────┘
                   │ ssh -L 5432:127.0.0.1:5432 -p 234
                   │   gbz20@49.213.71.8
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│ VPS 49.213.71.8                                                  │
│   127.0.0.1:5432  ← 生产 PostgreSQL                              │
└─────────────────────────────────────────────────────────────────┘
```

**三个关键坑**:

1. 容器里 PG_HOST **必须**是 `host.docker.internal`,不能是 `127.0.0.1`/`localhost`(那是容器自己)。
2. 宿主机 5432 **必须被 ssh.exe 占用**。如果本机装了 PostgreSQL 也会占 5432,SSH 隧道会绑定失败 —— 而且登录照常成功、**不报错退出**,极容易误判「隧道起来了」。所以下一步有专门的验证命令。
3. 隧道那个终端 **不能 `exit` / 不能关窗**,断了容器立刻连不上 PG。

## 步骤

### 1. 开 SSH 隧道到生产 PG(单独一个终端,保持开着)

```bash
ssh -L 5432:127.0.0.1:5432 -p 234 gbz20@49.213.71.8
```

容器启动时 `init_auth_db` / `init_analytics_db` 会立刻连 PG,**隧道没起容器直接崩**。

**验证隧道真的起来了**(另开一个 MINGW64 终端,不要碰 SSH 那个):

```bash
# 1. 确认 5432 在监听,记下最右列的 PID(下面示例是 16984)
netstat -ano | grep 5432
#   TCP    127.0.0.1:5432   0.0.0.0:0   LISTENING   16984
#   TCP    [::1]:5432       [::]:0      LISTENING   16984

# 2. 确认这个 PID 是 ssh.exe 而不是本机 PostgreSQL
tasklist //FI "PID eq 16984"
#   期望:ssh.exe  16984  Console  ...  12,636 K
```

看到 `ssh.exe` 就稳了。如果看到 `postgres.exe`,说明本机 PG 抢占了 5432,SSH 隧道**根本没绑上** —— 停掉本地 PG,或者把 Makefile 第 55 行 + docker-compose 里的端口都换成 `15432` 之类未占用端口。

### 2. 准备 .env

```bash
cp .env.example .env
# 编辑 .env,至少填 PG_PASS(从生产 /home/stackops/wanso-backend/.env 抄)
```

只想看 `/docs` 和非 AI 接口的话,其他字段可以留空。

### 3. 起容器

```bash
docker compose up --build
```

第一次 build 大约 2–4 分钟(pip 装 opencc / psycopg2-binary 等)。后续启动几秒。

### 4. 访问

- Swagger UI:http://localhost:8000/docs
- 健康接口:http://localhost:8000/api/stats
- 餐厅列表(走 SSH 隧道读生产 PG):http://localhost:8000/api/restaurants?city=hcmc&limit=5

## 常见问题

| 现象 | 原因 |
|---|---|
| 容器启动报 `could not connect to server` | SSH 隧道没起或断了 |
| `/api/chat` 返回 500 | `QWEN_API_KEY` 没填 |
| `/api/auth/register` 写入失败 | 正常 —— 连的是生产 DB,不要真写脏数据 |
| 改了 `.py` 不生效 | 需要 `docker compose up --build` 重建 |

## 改代码后重启

```bash
# Ctrl-C 停掉,然后:
docker compose up --build
```

或者起一个带 `--reload` 的开发模式(挂载源码):

```bash
docker compose run --rm -v "$(pwd):/home/stackops/wanso-backend" \
  wanso-backend uvicorn server:app --host 0.0.0.0 --port 8000 --reload
```

## 不在本次范围

- Refresh worker 不容器化(需要 Playwright + chromium + `sf_browser_data/` 登录态,本地用不上)
- 不起本地 PG 容器(通过 SSH 隧道连生产)
- 没改任何 `.py` 代码(硬编码路径 `/home/stackops/wanso-backend/...` 靠 Dockerfile 的 `WORKDIR` 复刻)
