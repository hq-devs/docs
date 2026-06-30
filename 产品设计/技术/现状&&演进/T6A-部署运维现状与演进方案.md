# T6A — 部署运维现状与演进方案

> 本文是部署分发与运维设计文档（T6）的**配套现状与演进方案**，描述**部署运维现状（As-Is）→ 目标设计（To-Be）的推进路线**：现有构建体系 / 配置 / Wails 打包的梳理、现状缺口汇总，以及补齐运维能力的优先级、Docker 最小可行配置与规则包版本化方案。
> **目标部署/运维设计（服务器容器化、桌面二进制分发、可观测性、本机工程目录备份与迁移目标功能）见 `T6-部署分发与运维设计.md`**，本文不重复描述目标态本身，只描述「如何从现状走到目标」。
> 本文以代码库事实（`Makefile`、`config/config.json`、`desktop/wails.json` 等）为基准梳理现状；现状随实现演进，缺口清单与里程碑也应随之更新。
> 相关文档：T1 系统架构总览、`T1A-现状到目标架构演进方案.md`、T5 调试子系统设计。

---

## 目录

- [§1 部署运维现状（As-Is）](#1-部署运维现状as-is)
- [§2 差距与演进](#2-差距与演进)

---

## §1 部署运维现状（As-Is）

### §1.1 现有构建体系

`Makefile`（位于 `ai-point-table/`）当前定义四个目标：

```makefile
wire:    # cd cmd/server && go run wire/cmd/wire  → 生成依赖注入代码
build:   # wire 后编译三个二进制：bin/server、bin/generate、bin/eval
vet:     # go vet ./...
test:    # go test ./...
```

- 无交叉编译目标，无版本号注入（`-ldflags`）。
- 无 CI 流水线配置文件（无 GitHub Actions / GitLab CI）。
- 无 Docker 构建目标。

### §1.2 现有配置

`config/config.json` 包含所有运行参数：

```json
{
  "server_addr": ":8080",
  "grpc_addr": ":50051",
  "database_dsn": "./data/app.db",
  "output_dir": "./output",
  "max_upload_size_mb": 30,
  "batch_size": 5,
  "max_concurrent_llm": 10,
  "openai": {
    "base_url": "http://58.250.250.47:31008/v1",
    "api_key": "sk-9037...",   ← 密钥硬编码在文件中（高风险）
    "model": "gpt-5.5",
    "max_retries": 5
  },
  "device_overrides": {}
}
```

**现状问题**：
- API Key 明文硬编码在配置文件，存在泄露风险。
- 无环境变量覆盖机制。
- 路径均为相对路径，容器化后需要手动修改。

### §1.3 现有 Wails 打包

- `desktop/wails.json`：定义产品名称（`设备驱动点表智能工作台`）、当前版本 `0.1.0`、构建脚本（`node sync-frontend.mjs`）。
- `desktop/README.md`：记录了 `wails build` 命令和已验证的 macOS Apple Silicon 产物路径。
- 已验证产物：`build/bin/point-table-workbench.app/Contents/MacOS/设备驱动点表智能工作台`。
- 无 Windows 交叉编译 CI 流程，无版本号自动注入，无 `.dmg`/`.msi` 打包配置。

### §1.4 现状缺口汇总

| 能力 | 现状 |
|---|---|
| 容器化 | ❌ 无 Dockerfile，无 docker-compose.yml |
| Secret 管理 | ❌ API Key 明文在 config.json |
| 环境变量覆盖 | ❌ 无，修改配置需改文件重启 |
| CI/CD 流水线 | ❌ 无 |
| 监控/告警 | ❌ 无结构化日志，无指标暴露 |
| 版本管理 | ❌ 客户端版本固定 0.1.0，无自动检查更新 |
| 规则包分发 | ❌ 规则文件静态嵌入二进制，无动态更新 |
| SQLite WAL | ❓ 未确认是否启用 WAL 模式 |
| 桌面 Windows 打包 CI | ❌ 无 |
| 迁移指引 | ❌ 无文档 |

---

## §2 差距与演进

### §2.1 必须补齐的运维能力（优先级排序）

| 优先级 | 能力项 | 目标里程碑 | 工作量估计 |
|---|---|---|---|
| P0 | **Secret 安全**：将 `api_key` 从 `config.json` 移出，改为环境变量注入 | 立即（M1 前） | 0.5 天 |
| P0 | **环境变量覆盖**：在 `config` 包加载逻辑中加入 `os.Getenv` 覆盖层 | M1 | 0.5 天 |
| P0 | **Dockerfile + docker-compose.yml**：最小可行容器化（见 §2.2） | M1 | 1 天 |
| P1 | **结构化日志**：替换 `log.Printf` 为 `slog`（Go 1.21+ 标准库），加入 gRPC `request_id` interceptor 与 Gin 兼容面中间件 | M1 | 1 天 |
| P1 | **SQLite WAL 模式**：启动时执行 `PRAGMA journal_mode=WAL` 等优化 | M1 | 0.5 天 |
| P1 | **客户端版本接口**：服务端暴露 `MetaService.GetClientVersion`，客户端启动时由 Bridge 检查 | M2 | 1 天 |
| P1 | **规则包版本接口**：服务端暴露 RulePackService gRPC 方法，客户端按版本缓存 | M2 | 2 天 |
| P2 | **工程用量汇总 API**：提供工程级用量查询，不暴露 token/model 明细 | M2 | 1 天 |
| P2 | **Prometheus 指标**：接入 `client_golang`，暴露 `/metrics` | M2 | 1 天 |
| P2 | **CI 流水线**：GitHub Actions 实现 `wire → test → build → docker push` | M2 | 1.5 天 |
| P3 | **Grafana 看板 + 告警规则** | M3 | 2 天 |
| P3 | **桌面端 Windows MSI 打包** | M3 | 1 天 |
| P3 | **output_dir 自动清理任务**（清理 30 天以上产物） | M3 | 0.5 天 |

> 上述能力项的目标设计分别对应 T6 §1.2（容器化 / 配置 / WAL）、§1.3（版本与规则包分发）、§1.5（可观测性）。

### §2.2 Docker/Compose 最小可行配置示例

**目录结构**：

```
ai-point-table/
├── Dockerfile              ← 新增
├── docker-compose.yml      ← 新增
├── .env.example            ← 新增（模板，提交 git；不提交实际 .env）
├── config/
│   └── config.json         ← 移除 api_key，改为环境变量占位
└── ...
```

**最小 `.env.example`**：

```bash
# 复制为 .env.local 并填写真实值（.env.local 不提交 git）
OPENAI_BASE_URL=http://your-llm-gateway/v1
OPENAI_API_KEY=sk-your-key-here
OPENAI_MODEL=gpt-5.5
MINERU_API_BASE_URL=http://192.168.20.99:8001
MINERU_VLM_SERVER_URL=http://mineru-openai-server:30000
MINERU_TIMEOUT_MINUTES=30
MINERU_MAX_CONCURRENT=2
XBOARD_BASE_URL=http://your-xboard/
XBOARD_TOKEN=your-token-here
APP_OUTPUT_DIR=/app/output
APP_DATABASE_DSN=/app/data/app.db
VERSION=v0.1.0
```

**最小 `docker-compose.yml`**：

```yaml
version: "3.9"

services:
  server:
    build:
      context: .
      args:
        VERSION: ${VERSION:-dev}
    image: ai-point-table-server:${VERSION:-dev}
    env_file:
      - .env.local
    environment:
      APP_DATABASE_DSN: /app/data/app.db
      APP_OUTPUT_DIR: /app/output
    volumes:
      - server_data:/app/data
      - server_output:/app/output
	    ports:
	      - "50051:50051"
	      - "8080:8080"
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:8080/health || exit 1"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 15s
    restart: unless-stopped

volumes:
  server_data:
  server_output:
```

**`config/config.json`（移除硬编码密钥后的基准文件）**：

```json
{
  "server_addr": ":8080",
  "grpc_addr": ":50051",
  "database_dsn": "./data/app.db",
  "output_dir": "./output",
  "max_upload_size_mb": 30,
  "batch_size": 5,
  "max_concurrent_llm": 10,
  "openai": {
    "base_url": "",
    "api_key": "",
    "model": "gpt-5.5",
    "max_retries": 5
  },
  "device_overrides": {}
}
```

> `base_url` 与 `api_key` 留空，由环境变量 `OPENAI_BASE_URL` / `OPENAI_API_KEY` 在运行时覆盖。`config` 包加载时优先读取同名环境变量（若非空则覆盖 JSON 值）。

### §2.3 规则包版本化最小方案

1. **规则文件 git 化**：将 `point_table_rules.json`、`groups_brd.json` 纳入 `ai-point-table` 仓库 `internal/validator/data/` 目录下（当前已通过 `go:embed` 静态嵌入二进制）。
2. **版本文件**：在同目录增加 `rule_pack_version.json`：
   ```json
   { "version": "v2026.06.01", "updated_at": "2026-06-01T00:00:00Z" }
   ```
3. **服务端接口**：暴露 `RulePackService.GetRulePackVersion`（返回版本+校验和）和 `RulePackService.DownloadRulePack`（返回规则包内容）。
4. **客户端缓存**：将下载的规则包缓存在 `~/.config/point-table-workbench/rule-pack/`，按 `version` 命名；对比版本号决定是否需要更新。
5. **滚动更新**：规则包更新**不需要**重新构建/分发客户端二进制，仅需服务端部署新版本规则文件即可生效。

```mermaid
flowchart LR
    maintainer["规则维护者\n更新规则 JSON"] --> git["git commit + tag\nv2026.06.02"]
    git --> ci["CI 重新部署服务端\n(rules 已 embed 进新镜像)"]
    ci --> server["RulePackService\n返回新版本号"]
    server --> client["客户端下次启动\n检测到版本差异"]
    client --> update["静默下载+缓存\n状态栏显示已更新"]
```
