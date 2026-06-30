# T11 — Agent 能力包集成与扩展指南

> **文档定位**：两个核心能力包（`generation` / `debugging`）已落地封装。本文是面向**使用者与扩展者**的操作手册（playbook）：① **系统如何把这两个能力包接入**到任意后端；② **以后如何扩展**（新增一个字段推断家族成员、新增校验闸、扩充调试可修正字段、升级 Prompt、新增一个全新 Agent）。
> **与 T2/T10 的关系**：T2 是设计层（契约/原则/状态机，权威）；T10 是实现层（包结构/骨架，权威）；**T11 是"怎么用、怎么扩"**——只讲操作步骤与真实接口，不重复定义设计。
> **配套仓库**：本文所有接口名、构造签名、适配器均来自 `ai-point-table` 仓库实测代码（`internal/agentkit`、`internal/capability/{generation,debugging}`、`internal/adapters`、`cmd/*-harness`）。如代码与本文不一致，以代码为准并回头修订本文。

---

## 目录

- [§1 心智模型：能力包对外长什么样](#1-心智模型能力包对外长什么样)
- [§2 系统如何接入这两个能力包](#2-系统如何接入这两个能力包)
- [§3 如何扩展一个能力](#3-如何扩展一个能力)
- [§4 独立运行与验证（harness）](#4-独立运行与验证harness)
- [§5 接入 / 扩展 checklist](#5-接入--扩展-checklist)

---

## §1 心智模型：能力包对外长什么样

两个能力包对任意系统都呈现同一形态（T2 §3.10）：

```
能力 = 同步门面(ctx, In) → (Out, error)
     + 注入端口（LLM / OCR / ArtifactStore / Xboard / SessionStore / ProgressSink）
     + 稳定哨兵错误集（agentkit.Err*）
```

- **门面同步、可取消**：能力包不自起 goroutine、不自管 job/队列、不自做限流、不读 env/全局。
- **传输无关**：不 import gRPC/Wails/Gin；接入只是"实现端口 + 在传输层调门面"。
- **职责切分**：

| 能力包负责 | 系统负责 |
|---|---|
| 推理与编排（状态机、三道闸、证据、安全门、回滚） | 传输（gRPC）、异步/任务队列、设备串行锁、鉴权/多租户、配置、物理存储、进度传输 |

两个门面（实测签名）：

```go
// internal/capability/generation
type Generation interface {
    Generate(ctx context.Context, in GenerateInput) (GenerateResult, error)
    Clarify(ctx context.Context, runID string) ([]Clarification, error)
    ApplyAnswers(ctx context.Context, runID string, ans []ClarificationAnswer) (GenerateResult, error)
    AnalyzeDelta(ctx context.Context, baseRunID string, p ProtocolInput) ([]ChangeSetItem, error)
}

// internal/capability/debugging
type Debugging interface {
    Debug(ctx context.Context, in DebugInput) (DebugResult, error)
}
```

> **核心用法（两条主路径）**：
> - 生成是**两阶段交互**：`Generate`（产草稿 + 不确定项）→ `Clarify`（出候选选项）→ `ApplyAnswers`（用户选中项 fold 落定为最终 DSL，bump `dsl_version`）。疑虑点的选择是产最终点表的核心环节，非可选旁路。
> - 调试是**唯一形态的自收敛 loop**：`Debug` 内部 采集→诊断→自动锁定正确点→对剩余点提假设并自动应用（每轮 `/updateTemplate` 热重载、复用同一会话容器）→回采验证，直到收敛（`converged`）或达 `MaxRounds`（`partial`）。无 report-only、无人工 Decide/Apply。

---

## §2 系统如何接入这两个能力包

### 2.1 接入三步

```
① 实现端口（或直接用现成适配器）
② 装配构造门面：generation.NewService(Deps, Config) / debugging.NewService(Deps, Config)
③ 在传输层（gRPC handler）注入调用门面，把错误/进度翻译给系统
```

能力包内部**零改动**——接入全部发生在系统侧。

### 2.2 端口与现成适配器对照

| 端口 | 谁用 | 现成适配器 | 自定义时实现什么 |
|---|---|---|---|
| `agentkit.LLM` | 生成 + 调试 | `adapters/llmopenai`（OpenAI 兼容）、`adapters/llmreplay`（回放/桩） | `Generate(ctx, system, user, schema)` |
| `generation.OCR` | 生成 | `adapters/ocrmineru`（MinerU / `NewMarkdownReader` 直读 Markdown） | `Parse(ctx, SourceFile) (ProtocolInput, error)` |
| `agentkit.ArtifactStore` | 生成 | `adapters/artifactfs`（本地目录） | 检查点/产物读写（逻辑名，物理布局自定） |
| `debugging.Xboard` | 调试 | `adapters/xboardhttp`（在线）、`adapters/mockxboard`（离线回放） | Update/Status/CollectValue/GetDebug/Deploy/Restore |
| `debugging.SessionStore` | 调试（可空） | `adapters/sessionfs` | `Save/Load(debugID, Session)` |
| `agentkit.ProgressSink` | 生成 + 调试（可空） | 无（系统适配）；缺省 `agentkit.NopSink` | `Emit(ProgressEvent)` → gRPC 流 / EventsEmit / 日志 |
| `agentkit.PromptSet` | 生成 + 调试 | 各包 `DefaultPromptSet()`（内嵌 v1） | `Resolve(ref)`/`Version()` |

> 端口缺失校验：`NewService` 对必填端口（生成：LLM/Store/Prompts；调试：LLM/Xboard/Prompts）为 nil 时返回 `agentkit.ErrInvalidInput`；`Sink` 为 nil 自动落 `NopSink`。

### 2.3 接入生成能力（gRPC handler 内）

```go
// 装配（启动期一次）
genSvc, err := generation.NewService(generation.Deps{
    LLM:      llmopenai.New(openAIClient),   // 系统注入真实/路由 LLM
    OCR:      ocrmineru.NewMarkdownReader(), // 文档已由 B 域 OCR 解析；直连 MinerU 用 ocrmineru.New(baseURL, timeoutMs)
    Store:    artifactfs.New(jobsDir),       // 或对象存储/DB 适配器
    Sink:     grpcProgressSink,              // §2.5
    Prompts:  generation.DefaultPromptSet(), // 或注入自定义/版本钉死
    GroupIdx: groupIdx,                       // 可空：跳过 board_type 检索
}, generation.Config{ /* 全默认即可 */ })
if err != nil { /* 启动失败 */ }

// gRPC handler 调门面（GenerationService.Generate，对齐 T4 C-1）
func (h *GenerationHandler) Generate(ctx context.Context, req *ptwv1.GenerateRequest) (*ptwv1.GenerationRun, error) {
    res, err := h.gen.Generate(ctx, generation.GenerateInput{
        Protocol: generation.ProtocolInput{Text: req.GetProtocolText()},
        RunID:    req.GetTaskId(), // 或系统生成
    })
    if err != nil {
        return nil, mapErr(err) // §2.6：哨兵错误 → codes.*
    }
    return toProto(res), nil // GenerateResult → proto
}
```

### 2.4 接入调试能力

```go
dbgSvc, err := debugging.NewService(debugging.Deps{
    LLM:      llmopenai.New(openAIClient),
    Xboard:   xboardhttp.New(xboardClient, deployer, debugBaseURL, timeoutMs), // 或 mockxboard.NewFromLayout(...)（离线回放）
    Sessions: sessionfs.New(jobsDir),       // 可空
    Sink:     grpcProgressSink,
    Prompts:  debugging.DefaultPromptSet(),
}, debugging.Config{})

// gRPC handler（DebugService.StartDebug，对齐 T4 H-1）
res, err := dbgSvc.Debug(ctx, debugging.DebugInput{
    Target:  debugging.DebugTarget{ResourceID: rid, BoardType: bt, RunID: runID},
    Table:   debugging.PointTable{Points: pts, Layout: lo, DeviceInfo: di, Xlsx: xlsxBytes},
    Options: debugging.DebugOptions{LockedSeqs: locked, MaxRounds: 5}, // 唯一形态=自收敛 loop；无 AutoFix/report-only
})
// res.Status ∈ {converged, partial, failed}；partial 时 res.UnconvergedSeqs 为残留点
```

> **关键约束（T2 §3.10①）**：调试唯一形态=自收敛 loop（无 `AutoFix`/`Mode`/report-only，无人工 Decide/Apply），变更经安全门自动应用；每轮重部署走 xboard `/updateTemplate`（整个 loop 复用同一会话容器，见 T9 §3.1）。异步执行、设备串行锁、任务排队、会话容器生命周期都属系统层——能力包只暴露同步 `Debug`。系统应在调用前对 `ResourceID` 加串行锁，调用放进自己的 job/队列。

### 2.5 进度外推：ProgressSink → gRPC 流

能力包发结构化 `agentkit.ProgressEvent`；系统实现 `Sink` 把它转成任意传输。事件 JSON 形状以 **T4 §1.3** 为权威：

```go
type grpcProgressSink struct{ stream ptwv1.GenerationService_StreamProgressServer }

func (s grpcProgressSink) Emit(ev agentkit.ProgressEvent) {
    _ = s.stream.Send(&ptwv1.ProgressEvent{
        EventType: ev.Kind,  // stage_started|stage_done|round|verdict|frame|warning|done
        StageName: ev.Stage,
        // ev.Data 按 Kind 投影为 T4 §1.3 的负载
    })
}
```

- 生成进度 → `ptw:progress:{run_id}`；调试实时值/报文 → `ptw:realtime|frames:{debug_id}`（Bridge 经 EventsEmit 推前端，T8 §6）。
- Sink 只为"过程可见"，主结果仍由门面同步返回，二者不耦合；未注入时用 `NopSink`，不影响主结果。

### 2.6 错误映射：哨兵错误 → gRPC codes

能力包导出稳定哨兵错误（`internal/agentkit/errors.go`），系统用 `errors.Is` 映射，**不靠字符串匹配**。映射表与 **T4 §1.4 / T8 §5.4** 一致：

```go
func mapErr(err error) error {
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

| 哨兵错误 | gRPC code | 典型场景 |
|---|---|---|
| `ErrInvalidInput` | `InvalidArgument` | 端口缺失 / 入参非法 |
| `ErrNotFound` | `NotFound` | run / 资源不存在 |
| `ErrConflict` / `ErrBaseMismatch` | `Aborted` | 设备调试占用 / 基线变化 |
| `ErrNotDebuggable` | `FailedPrecondition` | run 不可调试 |
| `ErrXboardUnreachable` | `Unavailable` | xboard 不可达 |
| 其它 | `Internal` | 未归类 |

> `ErrGatesUnsatisfied`（三道闸有限次仍不过）由能力包**内部**降级处理（标不确定项），通常不冒泡到门面；若冒泡按 `Internal` 处理。

---

## §3 如何扩展一个能力

扩展是**加法**：实现统一契约 + 注册，编排骨架/控制器**零改动**。下面是最常见的 5 类扩展。

### 3.1 生成侧：新增一个字段推断家族成员（最常见）

场景：协议出现新维度（如「报警阈值」「死区」），要让 AI 推断该字段。在 `internal/capability/generation/fields/` 下：

**步骤①** 定义 `FieldKind` 与产物类型 + 结构闸 Schema（`field.go` / `types`）：

```go
// field.go
const Threshold FieldKind = "threshold"
// types: ThresholdView（投影输入）、ThresholdOutput{ Rows []ThresholdRow }、ThresholdRow{ PointID, ... , Evidence, Uncertainties }
```

**步骤②** 写投影 `Project`（只给该字段必要列，防 LLM 看到无关字段污染）：`types.ProjectThreshold(cp)`。

**步骤③** 写版本化 prompt 资产：`prompts/threshold/v1.tmpl`（内嵌，`DefaultPromptSet` 自动可解析）。

**步骤④** 实现 `FieldAgent` 并注册（复用现成 `genericFieldAgent` + `BaseAgent`，与 `agents.go` 同款）：

```go
// agents.go
func newThresholdAgent(llm agentkit.LLM, ps agentkit.PromptSet, maxRetries int) FieldAgent {
    desc := descOf("threshold", agentkit.TierLight)
    base := agentkit.NewBaseAgent(llm, agentkit.RunSpec[[]types.ThresholdView, types.ThresholdOutput]{
        Desc:   desc,
        Render: renderCandidates[[]types.ThresholdView](ps, desc.PromptRef, thresholdSystem),
        Parse:  parseJSON[types.ThresholdOutput],
        Gates:  []agentkit.Gate[types.ThresholdOutput]{agentkit.NewSchemaGate[types.ThresholdOutput](desc.OutputSchema)},
    }, maxRetries)
    return &genericFieldAgent[types.ThresholdView, types.ThresholdOutput]{
        kind: Threshold, base: base, desc: desc,
        applicable: func(cp types.CandidatePoint, _ ProjectCtx) bool { return !cp.SkipCandidate },
        project:    func(cp types.CandidatePoint, _ ProjectCtx) types.ThresholdView { return types.ProjectThreshold(cp) },
        toRows:     func(out types.ThresholdOutput, prov agentkit.Provenance) []FieldRow { /* out.Rows → []FieldRow */ },
    }
}
```

```go
// registry.go — 在 NewDefaultRegistry 里加一行注册
r.Register(newThresholdAgent(llm, ps, maxRetries))
```

**步骤⑤** 提供黄金集（输入/期望输出对）并纳入评估门禁（T2 §6 / T7）。

> 控制器 `INFER` 阶段只遍历 `Registry.All()`，新成员自动并发纳入；装配阶段按 `FieldRow.Field` 多路填入 `types.MergedPoint`，出处走 `RunEvidence`（`FieldEvidence` 以字段键索引，天然容纳新字段）。**无需改控制器、装配、证据链。**

### 3.2 给某字段加值域闸 / 语义闸（提高质量）

三道闸里结构闸已由 `NewSchemaGate` 覆盖；值域/语义闸是**领域逻辑**，按 Agent 定制。只需往该 Agent 的 `RunSpec.Gates` 追加：

```go
Gates: []agentkit.Gate[types.AddressOutput]{
    agentkit.NewSchemaGate[types.AddressOutput](desc.OutputSchema),     // ① 结构
    agentkit.NewValueGate[types.AddressOutput](func(o types.AddressOutput) error {
        // ② 值域：FC 码白名单、寄存器须在协议出现…
        for _, r := range o.Rows { if !validFC(r.FunctionCode) { return fmt.Errorf("非法功能码 %s", r.FunctionCode) } }
        return nil
    }),
    agentkit.NewSemanticGate[types.AddressOutput](func(o types.AddressOutput) error {
        // ③ 语义：跨字段一致性（parser 与 dataType 不矛盾…）
        return nil
    }),
}
```

失败时 `BaseAgent` 自动把闸的错误文本作为反馈，做有限次（`MaxRetries`）修复重试；仍失败降级为不确定项。无需改 `BaseAgent`。

### 3.3 调试侧：扩充可修正字段白名单

调试的 `PatchGuard` 只允许修改**白名单字段**（`internal/capability/debugging/fields.go` 的 `whitelist`），其余（序号/sheet/设备表等）一律禁改。新增一个可修正字段：

```go
// fields.go — whitelist 增加一项
"dead_zone": {Side: "read", Column: "死区", RenderKey: excel.FieldDeadZone,
    set: func(m *types.MergedPoint, v string) { m.DeadZone = v }},
```

`Repairer`（推理内核端口，T2 §5.8）产出的 `Hypothesis.Field` 命中白名单才会进入 patch；锁定点/已判正确点仍受 PatchGuard 保护、回采不降级仍是硬约束（T2 §5.5）。**安全门逻辑零改动。**

### 3.4 升级 Prompt 版本（不改代码逻辑）

Prompt 是版本化资产（`prompts/{name}/{version}.tmpl`，`agentkit.EmbedPromptSet`）。升级方式二选一：

- **整套升版**：新增 `prompts/*/v2.tmpl`，构造时注入 `agentkit.NewEmbedPromptSet(fs, "prompts", "v2")`，或门面 `GenOptions.PromptSetVersion="v2"` 钉死。
- **单 Agent 升版**：把该 Agent `Descriptor.PromptRef.Version` 指向新版本。

版本随 run 写入 `Provenance`（可复现）。**任何 Prompt/模型变更必须过黄金集评估门禁方可生效**（T2 §6）。

### 3.5 新增一个全新 Agent（非字段家族）

如新增 `ComprehendAgent` 之外的语义 Agent：直接用内核 `BaseAgent` + `RunSpec` 实现 `agentkit.Agent[In,Out]` 契约即可，由对应控制器在状态机的相应状态调用。模式与 §3.1 的 `newXxxAgent` 一致：声明 `Descriptor`（含 `OutputSchema`/`PromptRef`/`ModelTier`）→ `Render/Parse/Gates` → `NewBaseAgent`。内核负责"渲染→LLM→三道闸→修复重试→降级上抛"，你只写投影、解析与闸。

---

## §4 独立运行与验证（harness）

能力包无需等系统就绪即可端到端验证——`cmd/` 下的 harness 用现成适配器装配后直接跑：

| harness | 作用 | 关键装配 |
|---|---|---|
| `cmd/gen-harness` | 协议文本 → 点表，产物落目录 | `llmopenai`/`llmreplay` + `ocrmineru.NewMarkdownReader` + `artifactfs` + stdout Sink |
| `cmd/debug-harness` | 点表 + 设备 → 诊断/修复 | `llm*` + `xboardhttp`/`mockxboard` + `sessionfs` |

```bash
# 生成：真实 LLM
gen-harness --protocol proto.md --out ./gen-out --base-url <ep> --api-key <k> --model gpt-4o
# 生成：回放（无需联网，CI/黄金集回归）
gen-harness --protocol proto.md --out ./gen-out --replay resp.json
```

**两种确定性运行模式（无外部依赖）**：

- **回放 LLM**：`llmreplay.NewReplayFromFile(resp.json)` 重放固定响应；`llmreplay.NewStub()` 给最小合法桩 → 用于 CI / 黄金集回归（与模型解耦，结果可复现）。
- **离线 xboard**：`adapters/mockxboard` 回放采集值与报文 → 调试闭环在无真机时可跑。

> 评估门禁：每个 Agent 维护带标注的黄金集，调试侧维护带真值的现场样本（值+帧+正确判定）；Prompt/模型变更跑黄金集，关键指标回归超阈值**阻断生效**（T2 §6 / T7）。

---

## §5 接入 / 扩展 checklist

**系统接入两个能力包**

- [ ] 为每个端口选定适配器（现成 or 自实现）；必填端口非空（否则 `ErrInvalidInput`）
- [ ] `generation.NewService` / `debugging.NewService` 在启动期装配一次，注入 `Config`（全默认即可起步）
- [ ] gRPC handler 内调门面，`GenerateResult`/`DebugResult` → proto（对齐 T4 §1.2）
- [ ] 实现 `ProgressSink` 把 `ProgressEvent` 转 gRPC 流（事件格式对齐 T4 §1.3）
- [ ] 用 `errors.Is` + 哨兵错误做 code 映射（对齐 T4 §1.4 / T8 §5.4），不字符串匹配
- [ ] 异步执行、设备 `ResourceID` 串行锁、任务队列、鉴权/多租户、会话 xboard 容器生命周期在**系统层**实现（能力包不管）
- [ ] 调试唯一形态=自收敛 loop（无 report-only/AutoFix/人工 Decide-Apply）；变更经安全门自动应用；每轮重部署走 `/updateTemplate`

**扩展一个字段维度（生成）**

- [ ] 定义 `FieldKind` + 产物类型 + 结构闸 Schema
- [ ] 写 `Project` 投影（只给必要列）
- [ ] 写版本化 prompt（`prompts/{name}/v1.tmpl`）
- [ ] 实现 `FieldAgent`（`genericFieldAgent`+`BaseAgent`）并在 `NewDefaultRegistry` 注册
- [ ] 必要时加值域/语义闸（`NewValueGate`/`NewSemanticGate`）
- [ ] 提供黄金集，纳入评估门禁
- [ ] 控制器 / 装配 / 证据链 **不改动**（验证编译与 harness 通过）

**扩展调试可修正字段**

- [ ] `whitelist` 增加字段项（含 `RenderKey` 与 `set`）
- [ ] 确认 PatchGuard 保护（锁定/正确点）、回采不降级硬约束仍生效
- [ ] 补充该字段的假设/验证黄金样本
