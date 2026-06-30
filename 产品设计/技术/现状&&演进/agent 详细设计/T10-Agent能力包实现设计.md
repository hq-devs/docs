# T10 — Agent 能力包实现设计（实现层）

> **文档定位**：本文是「点表智能工作台」两大核心 AI 能力（生成 / 调试）的**实现层设计**。T2 定义「应该怎么设计」（契约、原则、状态机），**T10 给出「按 Go 代码怎么落地」**：包/目录结构、可编译的接口骨架、默认适配器、构造装配、Prompt 资产、测试 harness。本文命名与契约与 T2 一一对齐。
> **配套阅读**：T2（Agent 系统设计，契约权威）/ T3（数据与证据模型）/ T8（gRPC 桥接，对外传输）/ T9（调试报文链路）/ T7（Eval）。
> **配套仓库（权威来源）**：本文与 `ai-point-table` 仓库配套实现。文中引用但未展开的内容**以仓库代码为准**：领域类型（`internal/types`、`internal/debug`、`internal/layout`）、确定性 Tool（`merger`/`layout`/`excel`/`evidence`/`validator`/`groupindex`）、部署机制（`internal/debug/deployer.go`）、Prompt 正文（`internal/agents/prompts/`）、xboard HTTP 细节（`internal/xboard`）。本文负责架构、契约与命名，仓库负责被引用的实现。**唯一仓库与本文都未给、需实现者新写的**是：各 Agent 的输出 JSON Schema（结构闸）与值域/语义闸白名单（FC 码、parser 等），见 §3.2。
> **强约束继承自 T2 §3.10**：能力包同步、可取消、传输无关、配置全注入、寻址中立；本文所有骨架都遵守这五条。

---

## 目录

- [§1 实现层定位与 T2 的关系](#1-实现层定位与-t2-的关系)
- [§2 能力包的物理布局（包/目录）](#2-能力包的物理布局包目录)
- [§3 共享内核 agentkit 实现骨架](#3-共享内核-agentkit-实现骨架)
- [§4 端口与默认适配器实现](#4-端口与默认适配器实现)
- [§5 生成能力包实现](#5-生成能力包实现)
- [§6 调试能力包实现](#6-调试能力包实现)
- [§7 构造与装配（wiring）](#7-构造与装配wiring)
- [§8 独立运行 harness（cmd/）](#8-独立运行-harnesscmd)
- [§9 测试策略（golden / replay）](#9-测试策略golden--replay)

---

## §1 实现层定位与 T2 的关系

| 维度 | T2（设计层） | T10（实现层，本文） |
|---|---|---|
| 回答 | 应该怎么做：契约、原则、状态机、质量门 | 怎么写代码：包结构、接口骨架、适配器、装配、测试 |
| 产物 | `Agent` 抽象、三道闸、门面 IO/端口/证据契约 | 可编译的 Go 骨架、默认适配器、`cmd/` harness |
| 权威 | 契约与语义的唯一权威 | 落地形态的权威；与 T2 冲突时以 T2 契约为准 |

**实现层的两条铁律**（落实 T2 §3.8/§3.10）：

1. **两个能力包 = 两个自包含 Go 包**，对外只暴露门面接口 + 端口接口 + 哨兵错误；不 import 任何传输（gRPC/Wails/Gin）或基础设施（sqlite/具体对象存储）包。
2. **依赖倒置**：包内只依赖端口接口；真实实现（OpenAI、MinerU、xboard HTTP、本地文件）作为**适配器**放在能力包之外，由系统在装配时注入。

---

## §2 能力包的物理布局（包/目录）

module `go.xbrother.com/ai-point-table`，两个能力包及其依赖采用如下布局：

```
ai-point-table/
├── internal/
│   ├── agentkit/                 # 共享内核：Agent 抽象 / 三道闸 / 重试 / 遥测 / Prompt / ProgressSink / 哨兵错误
│   │   ├── agent.go              # Agent[In,Out]、Descriptor、Result
│   │   ├── gate.go               # 结构/值域/语义三道闸 + 校验驱动修复重试
│   │   ├── runner.go             # BaseAgent：把 LLM 调用 + 三道闸 + 遥测包成统一执行体
│   │   ├── prompt.go             # 版本化 Prompt 资产加载（embed + 版本号）
│   │   ├── progress.go           # ProgressSink / ProgressEvent（no-op 默认）
│   │   ├── errors.go             # 哨兵错误（ErrInvalidInput / ErrConflict / ...）
│   │   └── ports.go              # 跨能力共享端口：LLM / ArtifactStore
│   ├── capability/
│   │   ├── generation/           # 能力包①：协议 → 点表
│   │   │   ├── generation.go     # 门面 Generation 接口 + 实现 service
│   │   │   ├── controller.go     # GenerationController 状态机（COMPREHEND→...→GATE）
│   │   │   ├── fields/           # 字段推断家族：每个 FieldKind 一个文件 + 注册表
│   │   │   ├── io.go             # GenerateInput/Result、ProtocolInput 等门面 IO 类型
│   │   │   └── ports.go          # OCR 端口（其余复用 agentkit.LLM/ArtifactStore）
│   │   └── debugging/            # 能力包②：点表 + 设备 → 诊断与修正
│   │       ├── debugging.go      # 门面 Debugging 接口 + 实现 service
│   │       ├── controller.go     # DebugController 闭环状态机（DEPLOY→...→RESTORE）
│   │       ├── patchguard.go     # 写点安全门
│   │       ├── io.go             # DebugInput/Result、PointTable 等
│   │       └── ports.go          # Xboard / SessionStore 端口
│   └── adapters/                 # 端口的真实实现（能力包之外，可被系统替换）
│       ├── llmopenai/            # agentkit.LLM ← OpenAI 兼容服务
│       ├── ocrmineru/            # generation.OCR ← MinerU /file_parse
│       ├── xboardhttp/           # debugging.Xboard ← xboard HTTP（含 T9 GetDebug）
│       ├── artifactfs/           # agentkit.ArtifactStore ← 本地文件 jobfs
│       └── sessionfs/            # debugging.SessionStore ← 本地文件
└── cmd/
    ├── gen-harness/              # 独立跑生成能力包（无系统、无 gRPC）
    └── debug-harness/            # 独立跑调试能力包（连真实/录制 xboard）
```

> **依赖方向（必须单向）**：`adapters → capability/* → agentkit`，且 `capability/generation` 与 `capability/debugging` **互不依赖**。`cmd/*` 与未来的 gRPC handler 同属"系统层"，负责装配（注入适配器）+ 传输，能力包对它们零依赖。

---

## §3 共享内核 agentkit 实现骨架

两个能力共用一套 Agent 运行时。把 T2 §3.2 的契约落成可编译骨架。

### 3.1 Agent 抽象（`agentkit/agent.go`）

```go
package agentkit

import (
    "context"
    "encoding/json"
)

// Agent 是被控制器统一对待的封装单元（T2 §3.2）。In/Out 为投影视图与结构化产物。
type Agent[In, Out any] interface {
    Descriptor() Descriptor
    Run(ctx context.Context, in In) (Result[Out], error) // 返回前已过三道闸
}

type Descriptor struct {
    Name         string          // 稳定标识，遥测/出处 label
    Version      string          // 语义版本
    PromptRef    PromptRef       // 版本化 prompt 引用（§3.4）
    OutputSchema json.RawMessage // 输出 JSON Schema（结构闸）
    ModelTier    ModelTier       // 难度档 → 模型路由
}

type ModelTier string
const ( TierLight ModelTier = "light"; TierHeavy ModelTier = "heavy" )

type Result[Out any] struct {
    Value      Out
    Confidence map[string]float64
    Uncertain  []Uncertainty
    Provenance Provenance     // {Agent, PromptVersion, Model, Ts}
    Telemetry  CallTelemetry  // 延迟/重试/token/各闸失败计数
}

type CallTelemetry struct {
    LatencyMs        int64
    Retries          int
    PromptTokens     int
    CompletionTokens int
    GateFailures     map[string]int // structure|value|semantic → 次数
}
```

### 3.2 三道闸与校验驱动修复（`agentkit/gate.go`）

落地 T2 §3.3 的三道闸（结构/值域/语义）。闸是**可注入的策略**，便于按 Agent 定制：

```go
// Gate 对一次解析后的输出做一类检查；失败返回可反馈给模型的具体错误。
type Gate[Out any] interface {
    Name() string // "structure" | "value" | "semantic"
    Check(out Out) error
}

// StructureGate 用 OutputSchema 校验（json schema 库）。
// ValueGate 校验枚举/范围/引用完整性（如 FC 码白名单、寄存器须在协议出现）。
// SemanticGate 校验跨字段一致性（parser 与 dataType 不矛盾）。
```

### 3.3 BaseAgent 执行体（`agentkit/runner.go`）

所有具体 Agent 复用同一执行体：渲染 prompt → 调 LLM → 解析 → 依次过闸 → 失败带反馈重试 → 仍失败降级为不确定项上抛。

```go
type RunSpec[In, Out any] struct {
    Desc    Descriptor
    Render  func(in In) (system, user string, err error) // 投影 → prompt
    Parse   func(raw string) (Out, error)                // 解析 + uncertainties 字符串容错清洗
    Gates   []Gate[Out]                                  // 结构/值域/语义
    Extract func(Out) ([]Uncertainty, map[string]float64) // 抽不确定项/置信度
}

type baseAgent[In, Out any] struct {
    llm    LLM
    spec   RunSpec[In, Out]
    maxRetries int
}

func (a *baseAgent[In, Out]) Run(ctx context.Context, in In) (Result[Out], error) {
    var tel CallTelemetry; tel.GateFailures = map[string]int{}
    feedback := ""
    for attempt := 0; attempt <= a.maxRetries; attempt++ {
        if err := ctx.Err(); err != nil { return Result[Out]{}, err } // 响应取消（T2 §3.10①）
        sys, usr, err := a.spec.Render(in)
        if err != nil { return Result[Out]{}, fmt.Errorf("render: %w: %w", ErrInvalidInput, err) }
        if feedback != "" { usr += "\n\n[上次输出错误，请修正：" + feedback + "]" }
        raw, err := a.llm.Generate(ctx, sys, usr, a.spec.Desc.OutputSchema)
        if err != nil { return Result[Out]{}, fmt.Errorf("llm: %w", err) }
        out, perr := a.spec.Parse(raw)
        if perr != nil { feedback = "JSON 解析失败: " + perr.Error(); tel.Retries++; continue }
        if g, gerr := runGates(a.spec.Gates, out); gerr != nil {
            tel.GateFailures[g]++; feedback = gerr.Error(); tel.Retries++; continue
        }
        unc, conf := a.spec.Extract(out)
        return Result[Out]{Value: out, Uncertain: unc, Confidence: conf,
            Provenance: Provenance{Agent: a.spec.Desc.Name, PromptVersion: a.spec.Desc.PromptRef.Version, Ts: now()},
            Telemetry: tel}, nil
    }
    // 有限次仍失败：不静默通过，降级为不确定项上抛（由控制器决定标记/澄清）
    return Result[Out]{}, fmt.Errorf("gates unsatisfied after %d tries: %w", a.maxRetries, ErrGatesUnsatisfied)
}
```

> 失败语义：有限次仍不过闸不静默返回，而是"降级为不确定项"上抛，由控制器接管（生成→澄清队列，调试→现场验证）。

### 3.4 Prompt 资产与版本（`agentkit/prompt.go`）

Prompt 以 `//go:embed prompts/**/*.tmpl` 内嵌，**带版本维度**，使「Prompt/模型变更过评估门禁」（T2 §3.7/§6）可落地：

```go
type PromptRef struct { Name, Version string } // 例：{ "address", "v3" }

// 资产路径：prompts/{name}/{version}.tmpl；激活版本由注入的 PromptSet 决定（不读全局）。
type PromptSet interface { Resolve(ref PromptRef) (tmpl string, err error) }
```

- 资产按 `prompts/{name}/{version}.tmpl` 组织，多版本并存，黄金集（§9）按版本绑定。
- `GenOptions.PromptSetVersion` 决定整套激活版本，随 run 钉死写入 Provenance。

### 3.5 ProgressSink 与哨兵错误（`agentkit/progress.go` / `errors.go`）

直接落 T2 §3.10②③：

```go
type ProgressEvent struct { Kind, Stage string; Round int; Data any }
type ProgressSink interface { Emit(ev ProgressEvent) }
type NopSink struct{}; func (NopSink) Emit(ProgressEvent) {} // 未注入时默认

var (
    ErrInvalidInput      = errors.New("invalid input")        // → codes.InvalidArgument
    ErrNotFound          = errors.New("not found")            // → codes.NotFound
    ErrConflict          = errors.New("conflict")             // → codes.Aborted
    ErrNotDebuggable     = errors.New("run not debuggable")   // → codes.FailedPrecondition
    ErrBaseMismatch      = errors.New("base version mismatch")// → codes.Aborted
    ErrXboardUnreachable = errors.New("xboard unreachable")   // → codes.Unavailable
    ErrGatesUnsatisfied  = errors.New("output gates unsatisfied")
)
```

---

## §4 端口与默认适配器实现

端口定义在 `agentkit` 或能力包内（依赖倒置），真实实现在 `internal/adapters/`。每个端口都提供**默认适配器**，使能力包可脱离系统独立运行（T2 §3.8）。

### 4.1 端口清单与实现归属

| 端口 | 定义位置 | 真实适配器 | 默认/离线适配器（harness） |
|---|---|---|---|
| `LLM` | `agentkit/ports.go` | `adapters/llmopenai` | 录制回放（`replayLLM`） |
| `ArtifactStore` | `agentkit/ports.go` | `adapters/artifactfs`（jobfs） | 本地临时目录 |
| `OCR` | `capability/generation/ports.go` | `adapters/ocrmineru`（T4） | 预解析 Markdown 直读 |
| `Xboard` | `capability/debugging/ports.go` | `adapters/xboardhttp`（含 `GetDebug`/`Deploy`/`Restore`） | `mockxboard` / 录制回放 |
| `SessionStore` | `capability/debugging/ports.go` | `adapters/sessionfs` | 本地临时目录 |
| `ProgressSink` | `agentkit/progress.go` | 系统提供（gRPC stream / EventsEmit） | `NopSink` |

### 4.2 LLM 端口适配器

`LLM` 端口带可选 `schema`（结构化输出/结构闸）；支持 `json_schema` 的模型走结构化输出，不支持则忽略 `schema` 退化为纯文本 JSON：

```go
// agentkit/ports.go
type LLM interface {
    Generate(ctx context.Context, system, user string, schema json.RawMessage) (string, error)
}

// adapters/llmopenai/llmopenai.go
type Adapter struct{ cli *openaiClient; supportsSchema bool }
func (a *Adapter) Generate(ctx context.Context, sys, usr string, schema json.RawMessage) (string, error) {
    if a.supportsSchema && len(schema) > 0 { /* response_format = json_schema */ }
    return a.cli.chat(ctx, sys, usr)
}
```

### 4.3 Xboard 端口（对齐 T9）

调试能力包的 `Xboard` 端口聚合采集/调试所需的全部 xboard 交互；`GetDebug` 即 T9 报文，`Deploy/Restore` 负责点表热重载与回滚（**Deploy 实现 = 覆盖会话容器 `{board_type}.xlsx` + `POST /updateTemplate(board_type)` 驱逐缓存重载**，见 T9 §3.1 / T1 §2.3）：

```go
// capability/debugging/ports.go
type Xboard interface {
    Update(ctx context.Context, resourceIDs []string) error
    Status(ctx context.Context, resourceID string) ([]DeviceStatus, error)
    CollectValue(ctx context.Context, resourceID string) (CollectValue, error)
    GetDebug(ctx context.Context, resourceID, debugID string) (DebugLog, error)   // T9：收发原始帧
    Deploy(ctx context.Context, resourceID, effectiveXlsx string) (Baseline, error) // 覆盖 {board_type}.xlsx + /updateTemplate
    Restore(ctx context.Context, resourceID string, bl Baseline) error              // 重写旧 xlsx + /updateTemplate
}
```

> `DebugLog/DebugCommand/DebugPacket/DebugSpot` 结构与保活轮询节奏（30s/100 条约束）见 T9；`adapters/xboardhttp` 负责把 xboard HTTP（含 `GET /api/v3/project/device/debug`、`POST /updateTemplate`）翻译成该端口，能力包对 HTTP 无感。loop 每轮 `Deploy` 走 `/updateTemplate`（非 `/update`），重部署后等一个采集周期再采（T9 §3.4）。

### 4.4 ArtifactStore / SessionStore（寻址中立）

落实 T2 §3.10④：包内只用 `(runID, logicalName)`，物理布局（`jobs/{run_id}/gen/checkpoints/{state}.json`）是适配器细节：

```go
type ArtifactStore interface {
    PutCheckpoint(runID, state string, v any) error
    GetCheckpoint(runID, state string, v any) (found bool, err error)
    PutProduct(runID, name string, data []byte) error // merged.json / xlsx / evidence.json
}
```

---

## §5 生成能力包实现

兑现 T2 §4。门面同步、ctx 可取消、检查点可续跑。

### 5.1 门面（`generation/generation.go`）

```go
type Generation interface {                                  // T2 §3.9 门面
    Generate(ctx context.Context, in GenerateInput) (GenerateResult, error)
    Clarify(ctx context.Context, runID string) ([]Clarification, error)
    ApplyAnswers(ctx context.Context, runID string, ans []ClarificationAnswer) (GenerateResult, error)
    AnalyzeDelta(ctx context.Context, baseRunID string, p ProtocolInput) ([]ChangeSetItem, error)
}

type Service struct {
    llm     agentkit.LLM
    ocr     OCR
    store   agentkit.ArtifactStore
    sink    agentkit.ProgressSink
    prompts agentkit.PromptSet
    fields  *fields.Registry
    cfg     Config // 批大小/并发/默认模型档；全注入（T2 §3.10⑤）
}

func NewService(deps Deps, cfg Config) (*Service, error) { /* 校验端口非空，sink 缺省 NopSink */ }
```

### 5.2 控制器状态机（`generation/controller.go`）

状态机 `COMPREHEND → DISCOVER → INFER → ASSEMBLE → GATE → DONE|WARNINGS|NEEDS_CLARIFICATION`（T2 §4.1）。`NEEDS_CLARIFICATION` 是**两阶段生成的核心交互态**：`GATE` 有不确定项即进入，门面 `Clarify` 出候选选项、`ApplyAnswers` 把用户选中项确定性 fold 回填→`ASSEMBLE` 重算并 bump `dsl_version`（落定最终 DSL，对齐 T3）。每状态前查检查点（`input_hash` 一致即跳过），后落检查点：

```go
func (s *Service) Generate(ctx context.Context, in GenerateInput) (GenerateResult, error) {
    if err := validate(in); err != nil { return GenerateResult{}, err } // → ErrInvalidInput
    runID := orNewID(in.RunID)
    proto := in.Protocol
    if len(proto.Files) > 0 && proto.Text == "" {                 // OCR 端口（T2 §3.8）
        parsed, err := s.ocr.Parse(ctx, proto.Files[0]); if err != nil { return GenerateResult{}, err }
        proto = parsed
    }
    h := inputHash(proto, s.cfg, s.prompts.Version())             // 幂等键（T2 §4.1）
    st := s.loadOrInit(runID, h)
    for st.Stage != stageDone {
        if err := ctx.Err(); err != nil { return GenerateResult{}, err }
        s.sink.Emit(agentkit.ProgressEvent{Kind: "stage_started", Stage: string(st.Stage)})
        next, err := s.step(ctx, &st, proto)                      // 单步推进
        if err != nil { return GenerateResult{}, err }
        s.store.PutCheckpoint(runID, string(st.Stage), st)        // 寻址中立
        s.sink.Emit(agentkit.ProgressEvent{Kind: "stage_done", Stage: string(st.Stage)})
        st.Stage = next
    }
    return s.assembleResult(runID, st), nil
}
```

> `INFER` 状态内部即 T2 §4.4 的字段家族并发逻辑：`errgroup + semaphore + BatchSize`，遍历 `fields.Registry` 调度各字段 Agent。

### 5.3 字段推断家族注册表（`generation/fields/`）—— 落地关键

字段家族以「实现接口 + 注册」组织，新增字段维度零改控制器（T2 §4.4）：

```go
// fields/field.go
type FieldKind string
const ( Address FieldKind="address"; Parser FieldKind="parser"; Unit FieldKind="unit"
        StateMap FieldKind="state_map"; Naming FieldKind="naming"; WriteMeta FieldKind="write_meta"; BitField FieldKind="bit_field" )

type FieldAgent interface {
    Field() FieldKind
    Applicable(cp types.CandidatePoint) bool          // write_meta 仅 rw/w；mapping/naming 排除位域点
    Project(cp types.CandidatePoint) any              // 投影视图（防 LLM 看到无关字段）
    Descriptor() agentkit.Descriptor
    RunBatch(ctx context.Context, views []any) ([]FieldRow, error) // 输出过三道闸
}

// fields/registry.go
type Registry struct{ agents map[FieldKind]FieldAgent }
func (r *Registry) Register(a FieldAgent) { r.agents[a.Field()] = a }
func (r *Registry) All() []FieldAgent { /* 稳定顺序返回 */ }

// fields/address.go —— 地址与访问权限字段 Agent
type addressAgent struct{ base agentkit.Agent[[]types.AddressView, types.AddressOutput] }
func (a *addressAgent) Field() FieldKind { return Address }
func (a *addressAgent) Applicable(cp types.CandidatePoint) bool { return !cp.SkipCandidate }
func (a *addressAgent) Project(cp types.CandidatePoint) any { return types.ProjectAddress(cp, isBitField(cp)) }
func (a *addressAgent) RunBatch(ctx, views []any) ([]FieldRow, error) { /* 调 base.Run，映射为 FieldRow */ }
```

INFER 控制器对注册表无知具体字段：

```go
func (s *Service) infer(ctx context.Context, cands []types.CandidatePoint) ([]FieldRow, error) {
    sem := semaphore.NewWeighted(int64(s.cfg.MaxConcurrentLLM))
    g, gctx := errgroup.WithContext(ctx); var mu sync.Mutex; var rows []FieldRow
    for _, fa := range s.fields.All() {                              // 遍历注册表
        applicable := filter(cands, fa.Applicable)
        for _, batch := range chunk(applicable, s.cfg.BatchSize) {
            fa, batch := fa, batch
            g.Go(func() error {
                if err := sem.Acquire(gctx, 1); err != nil { return err }; defer sem.Release(1)
                out, err := fa.RunBatch(gctx, project(batch, fa))
                if err != nil { mu.Lock(); rows = append(rows, markUncertain(batch, fa.Field(), err)...); mu.Unlock(); return nil } // 部分失败降级
                mu.Lock(); rows = append(rows, out...); mu.Unlock(); return nil
            })
        }
    }
    return rows, g.Wait()
}
```

**新增字段维度 5 步**（对照 T2 §4.4）：① 定 `FieldKind` + `Value` 结构 + Schema；② 写 `Project`；③ 写 `prompts/{kind}/v1.tmpl`；④ 实现 `FieldAgent` 并 `Register`；⑤ 备黄金集（§9）。控制器/装配/Merge 均不改。

### 5.4 装配产物与证据

`ASSEMBLE` 由确定性 Tool 链完成：`merger.Merge` → `layout.Compute` → `excel.Write` → `evidence.Build`，产出 `[]types.MergedPoint` + `layout.Layout` + xlsx + `RunEvidence`。字段级出处统一进 `FieldEvidence`（T2 §3.8 决策②），不塞进 `MergedPoint`。

---

## §6 调试能力包实现

> **⚠️ 实现差距提示（目标态重构指引）**：本节为**目标态**设计。现有代码（`internal/capability/debugging/controller.go`、`io.go`）仍为旧模型——`DebugOptions.AutoFix`/`MaxIterations`、`ChangeSet`/`ProposedChange`/`Decision`/`awaiting_review`、采样轮+finalize 迭代等。需按本节重构为：自收敛 loop（无 AutoFix 开关）+ 可插拔 `Repairer` 端口（§6.5）+ 目标态 io 类型（§6.4）。本次仅改设计文档，不动代码。

兑现 T2 §5。**唯一形态=自收敛 loop、全程可回滚、设备级串行**（无 report-only、无人工 Decide/Apply）。异步提交与设备串行锁属于**系统层**编排（T2 §3.10①），不进能力包；能力包只暴露同步 `Debug`。loop 内变更经安全门自动应用；每轮重部署走 xboard `/updateTemplate`（覆盖 `{board_type}.xlsx` + 驱逐缓存重载，见 T9 §3.1 / T1 §2.3）。

### 6.1 门面（`debugging/debugging.go`）

```go
type Debugging interface {
    Debug(ctx context.Context, in DebugInput) (DebugResult, error) // 同步、可取消；异步由系统包一层
}

type Service struct {
    llm      agentkit.LLM
    xb       Xboard            // 含 GetDebug/Deploy/Restore（§4.3）
    store    SessionStore
    sink     agentkit.ProgressSink
    guard    *PatchGuard
    repairer Repairer          // 可插拔推理内核端口（§6.5）；v1 singleShot / v2 agentic
    cfg      Config
}
```

> 设备锁、`go func` 提交、job 状态在系统层（如 gRPC service）实现；能力包不再 import 锁与会话调度。`ErrConflict` 用于"设备调试占用"，由系统层在拿锁失败时返回。

### 6.2 控制器闭环（`debugging/controller.go`）

落地 T2 §5.1：`DEPLOY → OBSERVE → DIAGNOSE → (HYPOTHESIZE → ACT → VERIFY)* → FINALIZE → RESTORE`。骨架（`defer Restore` 兜底回滚加载位）：

```go
func (s *Service) Debug(ctx context.Context, in DebugInput) (DebugResult, error) {
    if err := validate(in); err != nil { return DebugResult{}, err }
    // Deploy = 覆盖会话容器 {board_type}.xlsx + POST /updateTemplate（驱逐缓存重载，T9 §3.1）；loop 全程复用同一会话容器
    bl, err := s.xb.Deploy(ctx, in.Target.ResourceID, in.PointTable.XlsxRefOrRender())
    if err != nil { return DebugResult{}, fmt.Errorf("%w: %v", agentkit.ErrXboardUnreachable, err) }
    defer s.xb.Restore(context.Background(), in.Target.ResourceID, bl) // 收工回滚加载位（铁律）

    converged := newSeqSet(in.LockedSeqs...) // 收敛锁定集：用户预锁定点 + 逐轮自动锁定的正确点（棘轮）
    var rounds []RoundResult
    for r := 1; r <= maxRounds(in); r++ {
        if err := ctx.Err(); err != nil { return DebugResult{}, err }
        obs, err := s.observe(ctx, in)                  // 采全表（含收敛点，供回归）：值 + T9 GetDebug 收发帧
        if err != nil { return DebugResult{}, err }      // 全失败 → ErrXboardUnreachable
        verdicts := s.diagnose(ctx, obs)                 // DiagnoseAgent → []Verdict（值+帧+协议）
        converged.addCorrect(verdicts)                   // ① 自动锁定本轮正确点，移出后续目标（棘轮）
        s.sink.Emit(agentkit.ProgressEvent{Kind:"round", Round:r, Data: verdicts})
        s.sink.Emit(agentkit.ProgressEvent{Kind:"converged", Round:r, Data: converged.slice()})
        rounds = append(rounds, RoundResult{Round:r, Verdicts:verdicts, ConvergedSeqs: converged.slice()})
        targets := nonConverged(verdicts, converged)     // 仅非收敛的怀疑/错误点
        if len(targets) == 0 { break }                   // ② 全表收敛 → 退出
        hyps, _ := s.repairer.Propose(ctx, targets, evidenceOf(obs), rejected) // 端口：只提议；v1/v2 换实现外壳零改动
        applied := false
        for _, h := range rank(hyps) {
            patch := s.guard.Filter(h, converged)        // 收敛集（含用户锁定）一律拦截
            if patch.Empty() { continue }
            snap := s.xb.snapshot()
            s.xb.actuate(ctx, patch)                     // 渲染新表 + /updateTemplate 热重载；等一个采集周期
            vr := s.verify(ctx, in, converged.slice())   // 回采；不降级基准=收敛集
            if vr.Passed && !vr.Degraded { applied = true; break } // 不降级才采纳
            s.xb.restore(snap); rejected.add(h)          // 否则回滚（重写旧表 + /updateTemplate），试下一假设
        }
        if !applied { break }                            // ③ 无更多假设可试 → 残留转 partial
    }
    return s.finalize(in, rounds, converged), nil        // converged 全覆盖→converged，否则 partial(附 UnconvergedSeqs)
}
```

### 6.3 PatchGuard（`debugging/patchguard.go`）

`PatchGuard` 是能力包内的确定性安全门（T2 §5.5）：**收敛锁定集**（用户预锁定点 + 逐轮自动锁定的正确点）的 `FieldDiff` 一律拒绝进入 patch；`Verify` 比对收敛集（`PrevCorrectSeqs`），任一由正确变非正确 → `Degraded=true` → 拒绝该假设并回滚（重写旧表 + `/updateTemplate`）。

### 6.4 数据契约对齐

`Verdict / Hypothesis / VerifyResult / RoundResult / AppliedChange / ApplyResult` 为 `debug` 包的权威类型（T2 §3.9 类型列表），其中 `Verdict` 含 `RequestHex/ResponseHex/ModbusExcept/CrcOK`（来自 T9）与 `Locked/LockSource`；`RoundResult` 含 `ConvergedSeqs`；`DebugResult.Status ∈ {converged, partial, failed}`（无 `awaiting_review`）。无 `ChangeSet`（人工评审产物已删），改为 `AppliedChange` 自动应用留痕。

目标态 `io.go` 门面类型（取代旧 `AutoFix`/`ChangeSet`/`awaiting_review` 模型）：

```go
// 入参
type DebugInput struct {
    Target     DebugTarget    // {ResourceID, RunID}
    PointTable PointTable     // effective DSL（含 board_type / xlsx 渲染）
    LockedSeqs []int          // 用户预锁定点（进收敛集，PatchGuard 保护）
    Options    DebugOptions
}
type DebugOptions struct {
    MaxRounds   int // 自收敛 loop 轮数预算（删除旧 AutoFix / MaxIterations）
    IntervalMs  int // 采样间隔（OBSERVE）
    SampleCount int // 每轮每点采样数
}

// 状态：自收敛三态（删 completed / awaiting_review）
type DebugStatus string
const (
    DebugConverged DebugStatus = "converged" // 全表收敛
    DebugPartial   DebugStatus = "partial"   // 达 MaxRounds 仍有残留
    DebugFailed    DebugStatus = "failed"    // 不可恢复错误（如 xboard 不可达）
)

// 出参
type DebugResult struct {
    Status          DebugStatus
    Rounds          []RoundResult   // 每轮诊断/应用留痕
    ConvergedSeqs   []int           // 终态收敛集
    UnconvergedSeqs []int           // 残留（partial 时非空）
    FinalDSLVersion string          // 收敛后落定的 dsl_version
}

// 单条自动应用留痕（替代旧 ChangeSet/ProposedChange/Decision；无人工 accept/reject）
type AppliedChange struct {
    Round        int
    Seq          int
    Field        string
    OldValue     string
    NewValue     string
    HypothesisID string
    Source       string // 固定 "auto_fix"
    Reason       string
    Evidence     FieldEvidence
}
```

> 删除项：`DebugOptions.AutoFix`/`MaxIterations`、`ChangeSet`/`ProposedChange`/`Decision`/`KindReview`/`Accepted()`、`Session.AutoFix`/`Session.ChangeSet`、`DebugStatus.completed`/`awaiting_review`。新增项：`DebugOptions.MaxRounds`、`Verdict.LockSource∈{user, auto_converged}`、`Session.ConvergedSeqs`/`UnconvergedSeqs`。

### 6.5 推理内核 `Repairer` 端口（落地 T2 §5.8）

控制器（§6.2）只依赖 `Repairer` 端口；v1/v2 切换实现而外壳（loop/PatchGuard/Deployer/Verifier/收敛判定）零改动。

```go
// debugging/repairer.go —— 外壳唯一依赖的推理内核端口
type Repairer interface {
    // 入参=本轮非收敛目标集 + 证据 + 已拒假设；出参=候选假设（只提议：不部署/不锁定/不判收敛）
    Propose(ctx context.Context, targets []FixInput, ev Evidence, rejected map[string]struct{}) ([]Hypothesis, error)
}

type FixInput struct { // 限定本轮可改对象，禁止触碰收敛集
    Seq           int
    PointName     string
    Verdict       Verdict     // level/confidence/suspect_fields/帧
    AllowedFields []FieldKind // 仅 suspect_fields，收敛点字段不在内
}
```

**v1 `singleShotRepairer`（M3，先做；永久基线/兜底）**：

```go
type singleShotRepairer struct{ agent *HypothesizeAgent } // 包现有单次 LLM 推理
func (r *singleShotRepairer) Propose(ctx, targets []FixInput, ev Evidence, rejected map[string]struct{}) ([]Hypothesis, error) {
    return r.agent.Run(ctx, hypoProjection(targets, ev, rejected)) // 一次推理出分级假设
}
```

**v2 `agenticRepairer`（M4，演进；同端口第二实现）**：端口内部跑只读侦查工具循环，外壳无感。

```go
type agenticRepairer struct{ llm agentkit.LLM; tools ToolRegistry; budget int }
func (r *agenticRepairer) Propose(ctx, targets []FixInput, ev Evidence, rejected map[string]struct{}) ([]Hypothesis, error) {
    // tool-loop：模型自主调只读侦查工具，最后经 propose_patch 产出假设；预算/工具白名单兜底
    // 不部署、不锁定、不判收敛——这些仍由外壳执行
}

// 只读侦查工具端口草案（零副作用、不消耗部署预算）
type ToolRegistry interface {
    ReadProtocol(section string) (string, error)              // 读协议片段
    InspectFrame(seq int) (FrameView, error)                  // 收发帧/CRC/异常码
    DecodeTry(seq int, opt DecodeOpt) (float64, error)        // 本地试算候选解码（纯计算，不碰设备）
    GetSamples(seq int) ([]string, error)                     // 已采工程值
}
// 唯一“副作用”工具 propose_patch(field_diffs)：仍只是提议，真正 ACT/部署/VERIFY/LOCK/收敛由外壳执行并过安全门。
```

> 升级 v2 的前置门是录制-回放评测台（T7 §1.6）：v2 须在回放台上不劣于 v1 基线方可上真机。`agenticRepairer` 跑飞/超预算时回退 v1。

---

## §7 构造与装配（wiring）

能力包**只在构造期接收端口**（依赖注入），不读 env/全局（T2 §3.10⑤）。两种装配场景：

### 7.1 系统装配（gRPC handler 内，生产）

```go
// 系统层（cmd/server 或 gRPC service）—— 唯一读 env、选传输、管异步的地方
llmPort  := llmopenai.New(openaiCfgFromEnv())
ocrPort  := ocrmineru.New(mineruCfgFromEnv())
artStore := artifactfs.New(jobfsRoot)
xbPort   := xboardhttp.New(xboardBaseURL)
sessStore:= sessionfs.New(jobfsRoot)

gen := generation.NewService(generation.Deps{LLM: llmPort, OCR: ocrPort, Store: artStore,
        Sink: grpcSink /* 把 ProgressEvent → gRPC server stream */, Prompts: promptSet}, genCfg)
dbg := debugging.NewService(debugging.Deps{LLM: llmPort, Xboard: xbPort, Store: sessStore,
        Sink: grpcSink}, dbgCfg)

// gRPC handler：异步/任务/鉴权/设备锁在这里，门面只被同步调用
func (h *Handler) Generate(req *pb.GenerateReq, stream pb.Gen_GenerateServer) error {
    out, err := h.gen.Generate(stream.Context(), toInput(req)) // 同步门面
    if err != nil { return mapSentinel(err) }                  // 哨兵 → codes.*
    return stream.Send(toProto(out))
}
```

> 传输是 gRPC（T8），但 `generation`/`debugging` 包对此零感知——它们只看到 `ctx`、端口、`ProgressSink`。

### 7.2 错误映射（哨兵 → gRPC codes）

```go
func mapSentinel(err error) error {
    switch {
    case errors.Is(err, agentkit.ErrInvalidInput):      return status.Error(codes.InvalidArgument, err.Error())
    case errors.Is(err, agentkit.ErrNotFound):          return status.Error(codes.NotFound, err.Error())
    case errors.Is(err, agentkit.ErrConflict),
         errors.Is(err, agentkit.ErrBaseMismatch):      return status.Error(codes.Aborted, err.Error())
    case errors.Is(err, agentkit.ErrNotDebuggable):     return status.Error(codes.FailedPrecondition, err.Error())
    case errors.Is(err, agentkit.ErrXboardUnreachable): return status.Error(codes.Unavailable, err.Error())
    default:                                            return status.Error(codes.Internal, err.Error())
    }
}
```

---

## §8 独立运行 harness（cmd/）

证明"先封装、后接入"：在**没有系统、没有 gRPC**时端到端跑通两个能力包（T2 §3.8 独立运行）。

### 8.1 `cmd/gen-harness`

```
gen-harness --protocol ./samples/foo.md --out ./out --prompts v1
```
- 装配：`LLM=llmopenai`（或 `--replay ./fixtures/llm` 回放）、`OCR=预解析直读`、`Store=artifactfs(临时目录)`、`Sink=stdout`。
- 产出 `merged.json / *.xlsx / evidence.json`，可直接喂 §9 黄金集比对。

### 8.2 `cmd/debug-harness`

```
debug-harness --resource 0_101 --table ./out/foo.xlsx --xboard http://127.0.0.1:6100 --rounds 2
```
- 装配：`Xboard=xboardhttp`（或 `cmd/mockxboard` / `--replay` 录制回放）、`LLM` 同上、`SessionStore=sessionfs(临时)`。
- 自收敛 loop 自动跑到收敛（受 PatchGuard + 不降级约束）；`--rounds` 设 `MaxRounds`，残留产 `partial`。每轮重部署经 `/updateTemplate`（mock 回放亦模拟该语义）。

> harness 与 gRPC handler 是**同一能力包的两个调用方**，证明传输无关；任何一个跑通即说明集成只剩"接线"。

---

## §9 测试策略（golden / replay）

| 层 | 对象 | 手段 |
|---|---|---|
| 单 Agent | 每个 FieldAgent / DiagnoseAgent | 黄金集（输入投影 → 期望输出），LLM 用录制回放，断言过三道闸 + 字段准确率（T7） |
| 三道闸 | `gate.go` | 构造非法输出（缺字段/越界/语义矛盾），断言被拦截并产可反馈错误 |
| 控制器 | 生成/调试状态机 | mock 端口 + 注入故障（某批失败 → 降级为不确定项，不崩） |
| 续跑/幂等 | 生成检查点 | 同 `input_hash` 重放产相同产物；中断后从检查点续 |
| 安全门 | PatchGuard | 锁定/正确点不被改、降级必拒回滚 |
| 端到端 | harness | replay 全端口跑完整流程，纳入 CI |

- **Prompt/模型变更门禁**（T2 §6）：变更后必须跑对应黄金集，关键指标回归超阈值则 CI 阻断。
- 录制回放夹具按 `PromptRef.Version` 归档，保证可复现。

> 集成到最终系统 = 实现端口适配器 + gRPC handler 调同步门面（§7）+ 注入配置，**能力包内部零改动**；harness 与 gRPC handler 是同一能力包的两个调用方（§8），互不影响。
