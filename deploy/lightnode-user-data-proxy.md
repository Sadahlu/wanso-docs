# LightNode 用户数据代理拆分

记录 `dev-gbz` 分支上两个连续 commit(`3cc6e89` → `359fbbb`)所做的运维变更。
这两个 commit 是一组:把**用户数据相关端点**从主 API 拆出来,反向代理到一台独立的 LightNode 后端。

| Commit | 时间 | 主题 |
|---|---|---|
| `3cc6e89` | 2026-07-10 14:50 UTC | `ops: route user data endpoints to LightNode` |
| `359fbbb` | 2026-07-10 23:59 +0800 | `fix: persist verified LightNode user data proxy` |

两个 commit 均以 `Wanso Technology` 团队身份署名 merge 进 `dev-gbz`。

---

## 背景:为什么拆

在这次变更之前,所有 `api.wanso.ai` 的请求都走同一台主 VPS(`49.213.71.8`)上的 gunicorn(`127.0.0.1:8000`)。

这两个 commit 把**涉及用户敏感数据**的路径从主 API 分流到一台独立后端 `user-api.wanso.ai`(LightNode,origin IP `130.94.3.235`)。

被拆分出去的端点(来自 `api.wanso.ai.conf` 中标记 `WANSO_USER_DATA_PROXY_TO_LIGHTNODE_V1` 的块):

| 路径 | 说明 |
|---|---|
| `^~ /api/auth/` | 登录注册、JWT、OAuth、邮箱验证(`auth.py`) |
| `^~ /api/analytics/` | 用户行为埋点(`analytics.py`) |
| `= /api/feedback` | chat 选品反馈 |
| `= /api/chat/feedback` | 同上,旧路径 |
| `= /api/crash/report` | 客户端崩溃上报 |

其余路径(餐厅、搜索、chat 推荐、菜单翻译、explore、insights 等)**仍走本地 gunicorn**。

---

## `3cc6e89` —— 架构落地

新增两个文件,建立拆分路由的框架。

### 1. `deploy/nginx/api.wanso.ai.conf`(新增,101 行)

完整的 nginx 站点配置,包含两个 server:

- **HTTP 80**(过渡期保留):老 app 仍走 `http://49.213.71.8` 直连,所有流量转发到 `wanso_backend` upstream。
- **HTTPS 443**(新增,Cloudflare 回源走这里):
  - 用 Cloudflare origin 证书(`/etc/ssl/cloudflare/api.wanso.ai.pem`)。
  - 标了 `WANSO_USER_DATA_PROXY_TO_LIGHTNODE_V1` 的 location 块,通过 `include /etc/nginx/snippets/wanso-user-data-proxy.conf;` 引用代理 snippet。
  - 其余 location 仍走 `wanso_backend` upstream。
  - 保留 `/avatars/` 静态文件 alias 和 `/health` 探针。

upstream 定义:
```nginx
upstream wanso_backend {
    server 127.0.0.1:8000 fail_timeout=0;
    keepalive 32;
}
```

### 2. `deploy/nginx/wanso-user-data-proxy.conf`(新增,20 行)

可复用的代理 snippet,被 HTTPS server 里 5 个 location 引用。初版内容:

```nginx
proxy_pass https://user-api.wanso.ai;

proxy_ssl_server_name on;
proxy_ssl_name user-api.wanso.ai;
proxy_ssl_verify on;
proxy_ssl_trusted_certificate /etc/ssl/certs/ca-certificates.crt;

proxy_set_header Host user-api.wanso.ai;
proxy_set_header Authorization $http_authorization;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host $host;

proxy_http_version 1.1;
proxy_connect_timeout 10s;
proxy_send_timeout 90s;
proxy_read_timeout 90s;
proxy_request_buffering off;
proxy_buffering off;
```

要点:
- `Authorization` 透传 —— 用户 JWT 直接交给后端做鉴权,nginx 不参与。
- 流式相关字段(`proxy_request_buffering off`、`proxy_buffering off`)关闭 —— 因为 `/api/feedback` / `/api/crash/report` 可能是 POST 大 body,`/api/auth/*` 在未来也可能 SSE。
- TLS 校验依赖系统 CA。

---

## `359fbbb` —— TLS 校验修复

`3cc6e89` 上线后,HTTPS→LightNode 的 TLS 校验**实际上跑不通**(系统 CA 不信任 LightNode origin 的证书链)。这个 commit 改了 snippet 并加了一张自签证书。

### 改动一:`wanso-user-data-proxy.conf`

```diff
- proxy_pass https://user-api.wanso.ai;
+ proxy_pass https://130.94.3.235;          # 直连 origin IP,绕开 DNS / Cloudflare 路径

  proxy_ssl_server_name on;
  proxy_ssl_name user-api.wanso.ai;
  proxy_ssl_verify on;
+ proxy_ssl_verify_depth 2;
- proxy_ssl_trusted_certificate /etc/ssl/certs/ca-certificates.crt;
+ proxy_ssl_trusted_certificate /etc/nginx/ssl/user-api-lightnode-origin.pem;

  ...
+ add_header X-Wanso-User-Proxy lightnode-origin-v2 always;   # 观察用,标识走到了 v2 路径
```

关键选择:
- **直连 IP** 而非域名 —— 避免 `user-api.wanso.ai` 解析到的证书链和 nginx 看到的不一致。
- **信任自签 origin 证书** 而非公共 CA —— LightNode origin 不在公共 CA 信任链里,改成直接信任具体那张证书。
- `proxy_ssl_verify_depth 2` —— 自签证书链深度浅,显式压到 2。
- 响应头 `X-Wanso-User-Proxy: lightnode-origin-v2` 便于排查时分清流量走的是哪条路径(v1 还是 v2)。

### 改动二:`deploy/nginx/user-api-lightnode-origin.pem`(新增,19 行)

LightNode origin 的公开证书,自签(CN = `user-api.wanso.ai`,10 年有效期)。
**注意:这只是公开的 origin cert,不是私钥** —— 私钥没在仓库里,提交是安全的。

部署路径:`/etc/nginx/ssl/user-api-lightnode-origin.pem`(snippet 里 `proxy_ssl_trusted_certificate` 指向的位置)。

---

## 数据流总览

变更后:

```
                    ┌──────────────────────────────────────────────────┐
                    │  api.wanso.ai (主 VPS 49.213.71.8,nginx)          │
                    │                                                  │
  Cloudflare ──HTTPS│──┐                                               │
                    │  ├─ /api/auth/    ┐                              │
                    │  ├─ /api/analytics/├─► snippet ──► https://130.94.3.235  (LightNode)
                    │  ├─ /api/feedback ┤    (TLS pin to self-signed     user-api.wanso.ai
                    │  ├─ /api/chat/feedback origin cert)               独立后端,处理用户敏感数据
                    │  └─ /api/crash/report┘                            │
                    │                                                  │
                    │  其余路径 ─────────────────► 127.0.0.1:8000       │
                    │                                  gunicorn(17 workers)
                    │                                  server.py 主 API
                    │                                                  │
  老 app ──HTTP─────│──► 同上                                           │
                    └──────────────────────────────────────────────────┘
```

---

## 后续要改这块时的注意事项

1. **加新的 user-data 路径**:同步更新 `deploy/nginx/api.wanso.ai.conf` 里 `WANSO_USER_DATA_PROXY_TO_LIGHTNODE_V1` 块的 location 列表。
2. **LightNode origin 证书 10 年后过期**(2026-06-26 起 10 年)—— 届时需要换证书并更新 `user-api-lightnode-origin.pem`。
3. **`proxy_pass` 写死了 IP `130.94.3.235`** —— LightNode 换 IP 时记得改。
4. CLAUDE.md(backend / wanso-docs)**没有提到这次拆分**,如果以后要扩展或回滚这套代理,这份文档是唯一记录。
5. 私钥**永远不要**提交进仓库。`.pem` 文件只允许是公开证书 —— 提交前确认 `BEGIN CERTIFICATE` 而非 `BEGIN PRIVATE KEY`。

---

## 涉及的文件清单

```
deploy/nginx/
├── api.wanso.ai.conf                    (新增,3cc6e89)
├── wanso-user-data-proxy.conf           (新增,3cc6e89;修改,359fbbb)
└── user-api-lightnode-origin.pem        (新增,359fbbb)
```

复现 diff:
```bash
git show 3cc6e89
git show 359fbbb
```
