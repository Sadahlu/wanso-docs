# `wanso/` 与 `wanso-backend/` 目录差异分析

> 对比对象:
> - `F:\Wanso\wanso` — VPS 上的完整工作副本(包含运行时配置与调试痕迹)
> - `F:\Wanso\wanso-backend` — 干净的 git 源码仓库
>
> 分析日期:2026-06-22

---

## 一、结论

`wanso-backend` 中"缺失"的文件**全部**都是被 `.gitignore` 明确排除的本地调试 / 补丁 / 测试类文件,**没有任何应该提交的文件缺失**。这是符合预期的状态。

---

## 二、缺失文件清单(按类别)

### 1. 调试脚本(`debug_*.py`)
- `debug_shopee_menu_api_body.py`
- `debug_shopee_network.py`

### 2. 探测脚本(`probe_*.py`)
- `probe_shopee_menu_context_request.py`
- `probe_shopee_not_found_strict.py`

### 3. 测试脚本(`test_*.py`)
- `test_shopee_api.py`
- `test_shopee_browser_fetch.py`
- `test_shopee_headful_batch.py`
- `test_shopee_headful_native_batch.py`
- `test_shopee_menu_parser_from_saved_response.py`
- `test_shopee_refresher_headful.py`

### 4. 补丁脚本(`patch_*.py`)
- `patch_geocode.py`
- `patch_place_v2.py`, `patch_place_v3.py`
- `patch_smartfallback.py` ~ `patch_smartfallback_v7.py`(共 8 个)

### 5. 本地工具 / 运行时配置(显式 gitignore)
- `.env` — 环境变量(**敏感**,绝不应提交)
- `refresh_config.json` — 运行时配置
- `fetch_shopee_assets_browser.py` — 本地 Shopee 抓取工具
- `grab_token_refresher.py` — Grab token 刷新器(Playwright)
- `server.py.broken_deleted_filter_20260613_134054` — 历史备份

---

## 三、对应的 `.gitignore` 规则

| 文件类别 | 匹配规则 | 来源 |
|---|---|---|
| `debug_*.py` | `debug_*.py` | `.gitignore:60` |
| `probe_*.py` | `probe_*.py` | `.gitignore:61` |
| `test_*.py` | `test_*.py` | `.gitignore:62` |
| `patch_*.py` | `patch_*.py` | `.gitignore:63` |
| `.env` | `.env` / `.env.*` / `*.env` | `.gitignore:11-13` |
| `refresh_config.json` | `refresh_config.json` | `.gitignore:45` |
| `fetch_shopee_assets_browser.py` | `fetch_shopee_assets_browser.py` | `.gitignore:75` |
| `grab_token_refresher.py` | `grab_token_refresher.py` | `.gitignore:76` |
| `server.py.broken_*` | `*.broken*` | `.gitignore:33` |

---

## 四、额外观察

### `server.py` 版本不一致

| 目录 | 大小 |
|---|---|
| `wanso-backend/server.py` | **503,812 字节**(更新) |
| `wanso/server.py` | 283,948 字节(旧版) |

`wanso-backend` 上的 `server.py` 是**更新的版本**,`wanso/` 里那份是旧版。这与目录定位一致:`wanso-backend` 是活跃开发仓库,`wanso/` 是较早同步出来的快照。

### 两个目录的定位

- `wanso/` — VPS 完整工作副本:源码 + 运行时配置 + 所有调试 / 补丁脚本的历史积累
- `wanso-backend/` — git 源码仓库:仅包含应当版本化的文件,所有本地噪声通过 `.gitignore` 排除

两者差异是**设计使然**,并非遗漏。
