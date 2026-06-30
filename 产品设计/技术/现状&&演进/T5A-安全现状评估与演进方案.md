# T5A — 安全现状评估与演进方案

> 本文是安全权限与审计设计文档（T5）的**配套演进方案**，描述**现状安全评估（As-Is）→ 目标安全体系（To-Be）的推进路线**：现有威胁全景、已有安全能力、鉴权链路现状与差距，以及按 P0/P1/P2 优先级与 M1/M2/M3 里程碑的补齐路线。
> **目标安全/鉴权/审计设计（gRPC Metadata 鉴权、写点安全门禁、审计留痕的目标态）见 `T5-安全权限与审计设计.md`**，本文不重复描述目标态本身，只描述「如何从现状走到目标」。
> 代码快照基准：`ai-point-table`（2026-06）、`ai-point-web`（2026-06）；现状随实现演进，威胁清单与差距也应随之更新。

---

## 目录

- [§1 现状安全评估](#1-现状安全评估)
- [§2 差距与演进](#2-差距与演进)

---

## §1 现状安全评估

### 1.1 威胁全景

| 威胁项 | 当前风险等级 | 影响 | 根因 |
|--------|-------------|------|------|
| **现有 HTTP API 无鉴权，23 个端点公开** | 🔴 严重 | 任何人可调用生成、调试、发布接口；数据完整性无保障 | `router.go` 无 Auth 中间件；目标态迁移到 gRPC Interceptor |
| **LLM API Key 明文存于 `config.json`** | 🔴 严重 | API Key 一旦泄漏（代码库/配置文件），攻击者可盗用 LLM 配额，产生巨额费用 | `config.json` 第 9 行：`"api_key": "sk-9037..."` |
| **无审计日志** | 🔴 严重 | 无法追溯谁改了点表、何时提交、写点操作无留痕，合规与事故分析无依据 | 无 `audit_log` 表，无 AuditService |
| **写点操作无权限控制** | 🔴 严重 | `POST /debug/:debug_id/apply` 任意调用可应用调试变更；无写点授权机制 | 无 write-auth 接口和中间件 |
| **无 TLS 强制** | 🟠 高 | 当前 Gin 以 HTTP 启动（`server_addr: ":8080"`）；目标 gRPC 若不开 TLS，Token 和点表数据仍会明文传输 | `config.json` + 无 TLS/gRPC 配置 |
| **无工程级数据隔离** | 🟠 高 | 工程师 A 可访问工程师 B 的 `run_id` 数据（越权访问） | 无 `workspace_id` 绑定与校验 |
| **平台 Token 无安全存储** | 🟡 中 | 原型用 `localStorage`（明文），正式版若沿用则 Token 易被窃取 | `app.go` 中 `GetConfig`/`SaveConfig` 为桩方法未实现 |
| **xboard 地址/凭证无保护** | 🟡 中 | xboard 地址在 debug 请求中明文传递，内网服务暴露 | `POST /runs/:run_id/debug` body 含 `xboard_addr` |
| **Job 文件无访问控制** | 🟡 中 | 服务器文件系统 `jobs/{run_id}/` 无路径遍历防护，API 路径参数需严格校验 | 缺少 `workspace_id` 归属检查 |

### 1.2 现有安全能力

| 能力 | 实现文件 | 说明 |
|------|----------|------|
| 设备调试串行锁 | `internal/debug/lock.go`（推测路径） | 防止同一设备被并发调试操控，是现有最有效的安全措施 |
| Panic 恢复 | `middleware/middleware.go: Recover()` | 防止 panic 泄漏内部错误信息 |
| 请求 ID 注入 | `middleware/middleware.go: RequestID()` | 为日志链路追踪提供基础，可扩展为审计相关字段 |
| 访问日志 | `middleware/middleware.go: AccessLog()` | 记录 Method/Path/Status/Duration，有助于安全事件排查 |
| LLM 调用指数退避重试 | `llm.Client` + `agents/base.go` | 防止 LLM 接口滥用导致配额耗尽的间接保护 |

### 1.3 鉴权链路现状 vs 目标

```
现状 Gin 中间件链：
Recover() → RequestID() → AccessLog() → [Handler 直接处理]

目标 gRPC 链路：
Unary/Stream AuthInterceptor() → WorkspaceGuard() → [gRPC Handler] → Service
                                                          ↓
                                             写点方法额外校验：
                                             WriteAuthGuard() → DebugService.SetPoint
```

### 1.4 LLM API Key 明文存储现状风险

**现状风险**（高危）：`config.json` 中明文存储 OpenAI API Key（`"api_key": "sk-903..."`），任何能读取该文件的人均可拿到密钥，无法追踪密钥被盗用。目标方案（环境变量注入 / 密钥管理服务 / 加密配置）见 `T5-安全权限与审计设计.md` §1.6。

---

## §2 差距与演进

### 2.1 安全能力补齐优先级清单

#### P0 — 必须（M1 上线前完成，否则不得开放外部访问）

| 编号 | 能力 | 代码改动点 | 预估工作量 |
|------|------|----------|----------|
| P0-1 | **LLM API Key 环境变量化** | `config/config.go`：`LoadAPIKey()` 优先读 `OPENAI_API_KEY` 环境变量；`config.json` 移除 `api_key` 字段；`.gitignore` 加入 `config.json`；Docker Compose 加 `env_file` | 0.5 天 |
| P0-2 | **服务器端 Bearer Token gRPC 鉴权拦截器** | 新增 `internal/grpc/interceptor/auth.go`（Unary/Stream Interceptor + `TokenVerifier` 接口）；gRPC Handler 从 context 读取 Claims；`TokenVerifier` 实现对接平台 D1 校验接口 | 2 天 |
| P0-3 | **gRPC/TLS 部署** | `cmd/server/main.go` 并行启动 gRPC TLS 与 Gin HTTP 兼容面；生产推荐由 Nginx/Caddy 统一做 TLS 终止或直接配置 gRPC TLS 证书 | 1 天 |

#### P1 — 重要（M2 阶段完成，多工程师并发使用前）

| 编号 | 能力 | 代码改动点 | 预估工作量 |
|------|------|----------|----------|
| P1-1 | **工程级数据隔离（workspace_id 绑定）** | SQLite `devices` / `run_id` 等表加 `workspace_id` 列；所有 Service 方法加 workspace 过滤；`assertWorkspace()` 工具函数；所有 Handler 接入 | 3 天 |
| P1-2 | **写点授权 Guard** | 新增 `DebugService.RequestWriteAuth`、`DebugService.SetPoint` gRPC 方法；新增 SQLite `write_auth` 表；`WriteAuthGuard` 校验 TTL + user + run 归属；`apply` 逻辑不受影响（只改 set-point） | 2 天 |
| P1-3 | **审计日志表与 AuditService** | SQLite 新增 `audit_log` 表（DDL 见 T5 §1.4）；新增 `internal/audit/service.go`（`Write()` 异步写入）；在 gRPC Handler / Service 关键操作埋点 | 2 天 |

#### P2 — 建议（M3 阶段完成，正式生产稳定后）

| 编号 | 能力 | 代码改动点 | 预估工作量 |
|------|------|----------|----------|
| P2-1 | **客户端 Token 加密存储** | `desktop/app.go`：`GetConfig()`/`SaveConfig()` 调用系统 Keychain（`go-keyring` 库）存取 Token；替换 `localStorage` 用法 | 1 天 |
| P2-2 | **审计查询接口** | 新增 `AuditService.ListRunAudit` gRPC 方法；`AuditStore` 查询方法（按 run_id / event_type / 时间段过滤） | 1 天 |
| P2-3 | **试运行间 TTL 自动销毁** | `DebugService` 增加实例 TTL 跟踪（内存 Map + 定时器）；TTL 到期时调用 xboard 销毁接口并记录审计；服务器重启时清理孤儿实例 | 2 天 |
| P2-4 | **客户端本机审计副本** | Wails `app.go` 新增 `AppendAuditLocal(deviceID, record string) error`；客户端在确认/提交/写点时追加写本地 `audit_local.jsonl` | 1 天 |

### 2.2 M1/M2/M3 安全基线

#### M1 安全基线（生成 GUI 化，无真实调试）

- ✅ LLM API Key 不出现在代码库和 `config.json`
- ✅ 服务器配置了 gRPC/TLS（TLS 1.2+）
- ✅ 所有桌面业务 gRPC 方法通过 Bearer Token Metadata 鉴权（无 Token 返回 `Unauthenticated`）
- ✅ 健康检查 `/health` 无需鉴权（监控探活）
- ⬜ 工程级隔离可暂缓（M1 单工程师场景，或临时以 Token 对应单一 workspace）
- ⬜ 审计日志可暂缓（M1 不涉及写点）

#### M2 安全基线（多工程师并发，快捷提交）

- ✅ 工程级 `workspace_id` 数据隔离
- ✅ 审计日志记录：AI 生成、澄清拍板、点表确认、快捷提交
- ✅ 客户端 Token 使用 Keychain 或 AES 加密存储（不明文存磁盘）
- ✅ 审计查询接口可用（`AuditService.ListRunAudit`）

#### M3 安全基线（真机调试闭环，写点功能）

- ✅ 写点授权机制完整（`RequestWriteAuth` + `SetPoint` + TTL 自动吊销）
- ✅ 所有写点操作记录审计日志（含 `write_auth_id`、寄存器地址、写入值）
- ✅ 设备代理隧道使用 WSS（TLS 1.2+）
- ✅ 试运行间 TTL 自动销毁机制运行
- ✅ 客户端本机审计副本（`audit_local.jsonl`）

### 2.3 最小可行实现路径（M1 P0 示例）

**三步完成 M1 P0 安全基线**：

**第一步**：清理 API Key 硬编码（30 分钟）

```bash
# 1. config.json 移除 api_key 字段
# 2. config.go 增加环境变量读取
# 3. .gitignore 追加 config.json
# 4. docker-compose.yml 增加 env_file 或 environment
```

**第二步**：添加 gRPC Auth Interceptor（1 天）

```go
// internal/grpc/interceptor/auth.go 新建
// internal/grpc/server.go 挂载 Unary/Stream Interceptor
// 临时实现：TokenVerifier 直接调用平台 D1 /auth/verify 接口（HTTP 调用）
```

**第三步**：gRPC/TLS 配置（半天）

```bash
# 方案 A：gRPC 原生 TLS（适合单机部署）
# grpc.NewServer(grpc.Creds(credentials.NewServerTLSFromFile(certFile, keyFile)))

# 方案 B：Nginx 反代（推荐生产）
# nginx.conf 配置 SSL/gRPC 终止 → upstream 转发到 :50051 gRPC；:8080 仅保留 xcmdb/health 内网面
```

完成上述三步后，产品达到**生产可接受的最低安全基线**，可向外部工程师分发使用。

---

*文档版本 V0.1 · 2026-06-18*
*作者：架构设计（T5A）*
*关联：T5（安全权限与审计设计）/ T1（系统架构）/ T3（数据模型）/ BRD §8 / 原型说明 F17/D6*
