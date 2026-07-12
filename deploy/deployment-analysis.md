# Wanso 后端部署方式分析

> 分析对象：`wanso.service`（systemd 服务）
> 分析日期：2026-06-14
> 结论：**裸机 / VM 直接部署（systemd + venv + gunicorn），无任何容器层**

---

## 一、判定依据

### 1. systemd 直接拉起 venv 里的 gunicorn

```ini
# /etc/systemd/system/wanso.service
ExecStart=/home/stackops/wanso-backend/venv/bin/gunicorn server:app \
  --worker-class uvicorn.workers.UvicornWorker \
  --workers 17 \
  --bind 127.0.0.1:8000 \
  ...
```

服务单元**直接调用虚拟环境的 gunicorn 二进制**，而不是 `docker run` / `docker compose up` / `podman` / `containerd`。

### 2. 进程树是 gunicorn master + 17 workers

- Main PID 是 gunicorn master
- 下面 17 个是 UvicornWorker 子进程
- 如果是容器，CGroup 里通常会看到 `docker` / `containerd` / `runc` 层，而不是直接是 Python 进程。

### 3. 日志写到宿主机文件系统

```
--access-logfile /var/log/wanso/access.log
--error-logfile  /var/log/wanso/error.log
StandardOutput=append:/var/log/wanso/stdout.log
StandardError=append:/var/log/wanso/stderr.log
```

直接落在宿主机目录，**不是容器 stdout**（容器化一般会让 gunicorn 输出到 `-` 然后由 docker / logging driver 收集）。

### 4. 绑定 127.0.0.1:8000

说明前面有反代（nginx / caddy 之类）也在宿主机上，**不是通过容器网络**。

### 5. 进程依赖声明

```ini
After=network.target postgresql.service
Requires=postgresql.service
```

PostgreSQL 同样直接装在宿主机上，Postgres 与应用同机、同 cgroup hierarchy，没有走容器编排。

---

## 二、当前部署形态总结

| 项 | 值 |
|---|---|
| 进程管理 | systemd (`wanso.service`, enabled) |
| 应用服务器 | Gunicorn + UvicornWorker, 17 workers |
| Python 环境 | venv at `/home/stackops/wanso-backend/venv` |
| 监听 | `127.0.0.1:8000`（本机，靠前置反代） |
| 数据库 | PostgreSQL（systemd 单元依赖，同机部署） |
| 日志 | `/var/log/wanso/{access,error,stdout,stderr}.log` |
| 资源限制 | `LimitNOFILE=65536` |
| 重启策略 | `Restart=always`, `RestartSec=5` |
| 容器 | **无** |

---

## 三、与容器部署的性能对比

> 对当前 workload（FastAPI + gunicorn + 17 个 uvicorn workers，Python 进程）
> **容器 vs 裸机的性能差距通常 < 1~3%**，因为瓶颈根本不在虚拟化层，
> 而在 Python 本身、数据库、网络 IO。

| 维度 | 裸机（systemd） | 容器（Docker） | 差异大小 |
|---|---|---|---|
| CPU 计算 | 直跑 | 多一层 namespace/cgroup | **< 1%**，几乎测不出 |
| 内存开销 | 仅 Python 进程 | + 容器 runtime 几十 MB | 可忽略 |
| 网络延迟 | 直绑 `127.0.0.1` | docker0 网桥 + iptables NAT，多一跳 | **1~5 µs/请求**，量小，但 P99 会变难看 |
| 磁盘读（加载模型/大文件） | 直接读 ext4 | overlay2 多一层联合，首次读稍慢 | 几 % |
| 磁盘写（日志文件） | 直接写 | overlay + 写时复制，append 场景基本无差 | 极小 |
| 冷启动 | `systemctl start` < 1s | 拉镜像 + 起 container，秒级 | 不影响稳态 |
| 进程间通信 | gunicorn master↔worker 走 socket，内核直连 | 走容器 netns，稍慢 | 极小 |

**稳态性能结论**：裸机略优，但量级小到正常业务根本感知不到。

---

## 四、真正的差别在「运维 / 工程」，不在「性能」

### 容器化的实际价值

1. **环境一致性** — 开发/测试/生产同一镜像，不会出现"我本地能跑"。
2. **部署可重现** — `docker pull` 即可回滚到任意版本；裸机回滚得靠 git checkout + 重装依赖 + restart。
3. **横向扩展** — 上 K8s / ECS 之前几乎必须先容器化。
4. **资源隔离更硬** — 容器 ns + cgroup 比纯 systemd cgroup 更彻底（挂了不影响宿主其他服务）。

### 裸机（systemd）的优势

1. **调试简单** — `journalctl`、`py-spy`、`strace`、直接 `gdb` 都能用；容器里要 `docker exec` 进去，符号/源码映射麻烦。
2. **少一层故障面** — 没有镜像、网络、storage driver 这些坑，出问题排查路径短。
3. **冷启动快** — systemd restart 几乎瞬时，容器重启要重新起进程。
4. **直接打文件日志** — `/var/log/wanso/access.log` 直接给日志采集/分析用，容器里要绕一圈 stdout。

---

## 五、当前配置评估

当前 unit 文件写得**很标准**，没有明显问题：

- `--workers 17` ≈ CPU 核数 × 2 ~ 3，合理（假设 8~16 核机器）
- `Restart=always`、`RestartSec=5` — 崩溃自愈
- `LimitNOFILE=65536` — 高并发必需
- 绑 `127.0.0.1:8000` + 前置反代 — 标准做法
- `After/Requires=postgresql.service` — 依赖声明清楚
- `WorkingDirectory` + `PATH` 都显式指定 — 环境干净

---

## 六、何时应该考虑容器化

**不需要为了"性能"而容器化**。仅当出现以下诉求时才值得迁：

- 多机器部署 / 弹性伸缩
- 想做蓝绿、金丝雀、版本回滚
- 想统一开发与生产环境
- 团队多人协作、想用 CI/CD 出镜像

**单机稳态服务、机器数量少**（比如就这台 wanso 主机），
**裸机部署反而是更省心的选择**。

---

## 附：100% 验证命令清单

```bash
sudo systemctl cat wanso        # 看 unit 文件到底 ExecStart 什么
which docker && docker ps        # 确认有没有 docker / 在跑的容器
ls /home/stackops/wanso-backend  # 确认是源码 + venv 直接放宿主机
```
