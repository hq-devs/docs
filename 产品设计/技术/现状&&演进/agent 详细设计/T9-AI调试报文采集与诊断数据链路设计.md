# T9 — 点表智能工作台：AI 调试报文采集与诊断数据链路设计

> **文档定位**：本文定义「点表智能工作台」**AI 调试阶段的报文采集与诊断数据链路**——AI 在真机调试时如何获取**带完整语义关联的原始收发报文**（原始帧 ↔ 命令 ↔ 测点 ↔ 解析值 ↔ 错误码），用于通讯层诊断（变比/字节序/异常码/超时）。
> **数据源**：调试期所有报文与判定数据，统一来自 xboard 的 `debug/info` 驱动调试接口（§2 为基于 `xboard.v2` 代码快照 2026-06 的权威契约）。
> **在能力架构中的位置**：报文链路是 T2 §5 调试能力闭环里 `OBSERVE` 阶段的数据来源——`Observer` 同轮采集「工程值 + 收发帧」组成 `RoundObservation`，喂给 `DiagnoseAgent`。本文命名与 T2（设计）、T10（实现）一一对齐。
> 关联文档：T1（系统架构）/ T2（Agent 设计，调试闭环）/ T10（能力包实现，Xboard 端口与 Observer）/ T4（H 域调试 UI）/ T8（gRPC Bridge 流式推送）。
> **配套仓库**：本文与 `ai-point-table` 仓库配套阅读。§2 为 `xboard.v2` 权威接口契约；后端 `Update`/`Status`/`CollectValue`/`Deploy`/`Restore` 等 xboard 交互的实现细节以 `internal/xboard`、`internal/debug` 为准。

---

## 目录

- [§1 设计目标与数据源](#1-设计目标与数据源)
- [§2 xboard debug/info 权威接口契约](#2-xboard-debuginfo-权威接口契约)
- [§3 调试闭环中的报文链路](#3-调试闭环中的报文链路)
- [§4 喂给 DiagnoseAgent](#4-喂给-diagnoseagent)
- [§5 与桌面端的衔接（T4 / T8）](#5-与桌面端的衔接t4--t8)
- [§6 与其他文档的对齐](#6-与其他文档的对齐)

---

## §1 设计目标与数据源

### 1.1 AI 调试准确性的本质依赖

AI 要正确诊断一张点表错在哪，必须**同时**握有两类信息，二者缺一即从「准确」退化为「猜测」：

| 维度 | 含义 | 为何不可缺 |
|---|---|---|
| **语义关联** | 一帧原始字节 ↔ 它属于哪条命令 ↔ 映射哪些测点 ↔ 解析成什么值 ↔ 驱动判定的错误码 | 没有关联，AI 无法把「`15 13` → 539.5A 超额定」归因到「A 相电流变比错误」这条具体结论 |
| **完整时序** | 收发帧成对、带时间戳、连续不丢 | 没有时序，无法判定超时、抖动、偶发 CRC 错 |

### 1.2 数据源：xboard debug/info（帧 + 完整语义绑定）

调试报文唯一取自 xboard 的 `debug/info` 驱动调试接口，因为它**天然输出「帧 + 含义」绑定**：

1. **语义关联只有 xboard 能正确计算，且不可在上层重建**。xboard 用点表驱动构帧、发帧、按点表解析响应，"这 14 字节 = 读命令1 = A/B/C 相电流 = 539.5A" 是它结合点表与协议算出来的。若只取裸帧流（如抓包文件），等于丢弃该关联，迫使 AI 用「一张待验证的点表」去重新解析裸帧再判断这张点表对错——逻辑自指、误差自放大。
2. `debug/info` 输出的「帧 + 命令 + 测点 + 解析值 + 错误码」结构化绑定，正是 AI 通讯层诊断与产品「收发报文」面板共同需要的形态。
3. xboard.v2 为权威实现，其按点表解析出的值与关联**视为可信**。

### 1.3 核心设计原则

- **数据源唯一**：调试期所有报文与判定数据，统一来自 xboard `debug/info`。
- **语义关联零丢弃**：后端透传 xboard 给出的「帧↔命令↔测点↔值↔错误码」绑定，不拆散。
- **在后端边界补齐连续性**：`debug/info` 是「调试态 + 轮询取走」的快照接口（见 §2.2 的 30 秒超时与 100 条上限），后端通过**保活轮询调度**（§3.4）在自收敛闭环的多轮采样节奏内稳定取帧，**不修改 xboard**。
- **统一锚点 seq**：报文按读表序号 `seq` 挂到 `Observer` 既有的 `seq ↔ spot ↔ point` 绑定上，与工程值共用同一关联，不引入第二套关联机制。
- **自收敛闭环为唯一形态**：调试是 T2 §5 的自收敛 loop（无 report-only、无人工 Decide/Apply）。每轮 `ACT` 重部署修改后的点表=**渲染新 xlsx 覆盖会话容器模板目录的 `{board_type}.xlsx` → `POST /updateTemplate([board_type])`**（驱逐 xboard 按 board_type 的内存模板缓存并重建板卡、重读新表）；**禁止只 `/update`**（缓存不驱逐→读旧表→假收敛/不收敛，机制见 T1 §2.3）。整个 loop **复用同一会话容器**，重部署后等一个采集周期再采。

---

## §2 xboard debug/info 权威接口契约

> 本节为基于 `xboard.v2` 代码的接口契约固化，作为后端 `xboardhttp` 适配器（T10 §4.3）实现 `Xboard.GetDebug` 的权威依据。

### 2.1 调用入口与调度链路

**对外 HTTP 接口（在线采集设备）**：

```
GET /api/v3/project/device/debug?resource_id=<设备ID>&debug_id=<调试会话ID>
```

来源：`xboard.v2/doc/doc/http-protocol.md`。调度链路：

```
HTTP /api/v3/project/device/debug
  → CollectApp::GetDebug                          (internal/app/collect/collect_app.cc)
    → agent->AdvancedRPC(resID, "debug/info", "", logs)
      → modbus AdvancedRPC("debug/info")          (drivers/modbus/modbus.cpp)
        → board->GetDebugLog(params, result)      (drivers/foundation/src/board.cpp)
```

`CollectApp::GetDebug` 固定以 `"debug/info"` 调用 Agent 的 AdvancedRPC：

```cpp
// internal/app/collect/collect_app.cc
int CollectApp::GetDebug(const ResourceDebugID* request, CommonMsg* response) {
  const std::string& resID = request->resource_id();
  auto agent = findAgentByResourceID(resID);
  ...
  int ret = agent->AdvancedRPC(resID, "debug/info", "", logs);
  ...
  response->set_data(logs);
  return 0;
}
```

> gRPC 等价入口：`Collect.GetDebug(ResourceDebugID) returns (CommonMsg)`（`internal/pkg/proto/collect/collect.proto`）。后端 `xboardhttp` 适配器默认走 HTTP，与 xboard 其余采集接口一致。

### 2.2 触发机制与生命周期（关键约束）

`debug/info` 是**调试态接口**，对一台正在采集的在线设备（`resource_id` 非 `d` 前缀），其行为由 `Board::GetDebugLog` 与 `Board::AddDebugLog` 决定：

**① 首次调用即「布防」**：若设备当前不在调试态，`GetDebugLog` 把该设备所有读命令置 `inDebug=true`，并记录 `debugBeginTime_`。此后采集执行器仅对 `inDebug` 的命令额外记录原始帧：

```cpp
// drivers/foundation/src/command_executor.cpp
if (command.inDebug && CommandRealDO(cmdSts)) {
  std::shared_ptr<CommandLog> cmdLog = std::make_shared<CommandLog>(command);
  brd->AddDebugLog(cmdLog);
}
```

**② 读取即清空（drain-on-read）**：每次 `GetDebugLog` 用 `logs.swap(debugLogs_)` 把累积日志取走并清空缓冲。

**③ 自动超时**：`AddDebugLog` 中若 `now - debugBeginTime_ > 30`（秒），自动将命令撤出调试态（停止记录）。

**④ 缓冲上限**：`debugLogs_` 超过 **100** 条时 `pop_front`（丢弃最旧）。

```cpp
// drivers/foundation/src/board.cpp  Board::AddDebugLog
if ((now < debugBeginTime_) || (now - debugBeginTime_ > 30)) {   // ③ 30s 自动停
  ... cmd->inDebug = false; ...
} else {
  if (debugLogs_.size() > 100) { debugLogs_.pop_front(); }        // ④ 100 条上限
  debugLogs_.push_back(log);
}
```

> **对后端的含义**：必须以 **远小于 30 秒** 的周期持续轮询（每次轮询既续命又取走数据），且轮询间隔内累积日志数须 < 100，才能把这个「快照 + 会超时」的接口用成「连续、无丢」的数据流。调度设计见 §3.4。

### 2.3 返回结构（在线设备，结构化格式）

在线设备（非 `d` 前缀）走 `GetDebugLog` 的结构化分支，返回每条命令的收发交互与解析测点：

```json
{
  "data": {
    "resource_id": "0_101",
    "debug_id": "1642730932",
    "version": "2.0",
    "commands": [
      {
        "command_id": 0,
        "error_code": 0,
        "error_msg": "success",
        "interact": [
          { "packet": "10:38:24.123 snd: 01 03 00 00 00 0A C5 CD" },
          { "packet": "10:38:24.182 rcv: 01 03 14 09 12 ..." }
        ],
        "spots": [
          { "name": "源1线电压AB", "resource_id": "0_101_1_1_0", "status": 1, "value": "232.2", "id": "1" }
        ]
      }
    ]
  },
  "error_code": "00",
  "error_msg": "success"
}
```

字段语义（来自 `Board::GetDebugLog` 结构化分支与 `CommandLog`）：

| 字段 | 含义 |
|---|---|
| `commands[].command_id` | 命令序号 |
| `commands[].error_code` / `error_msg` | 该命令收发状态（成功 / 发送失败 / 无响应 / 响应格式错误） |
| `commands[].interact[].packet` | 单条收发记录，格式 `"HH:MM:SS.mmm snd|rcv: <hex>"` |
| `commands[].spots[]` | 该命令解析出的测点：`name` / `resource_id` / `status` / `value` / `id` |

### 2.4 报文字符串格式与 hex 来源

`interact[].packet` 由 `CommandLog::LogRequest` / `LogResponse` 生成，**直接取自采集线程收发缓冲 `cmd.request_->buf` / `cmd.response_->buf`**，`hex` 默认开启（`CommandLog(Command& cmd, bool hex = true)`）：

```cpp
// drivers/foundation/src/command_log.cpp
std::string pkgTxt = fmt::format("{:%H:%M:%S}.{:03} snd: ", time, milli);   // 或 "rcv: "
if (hex_) {
  ...
  pkgTxt += fmt::format("{:02X}", fmt::join(vv, " "));   // 真实收发字节 → "01 03 00 00 00 0A C5 CD"
}
logPacket_.push_back(std::move(pkgTxt));
```

- 前缀 `snd:` = 请求帧（TX），`rcv:` = 响应帧（RX）。
- hex 为设备链路上的**真实原始字节**，Modbus 异常码可直接从响应帧解析（如 `rcv: 01 84 02`：`0x84 = 0x03|0x80` 表示 FC03 异常响应，`02` = 非法数据地址）。

---

## §3 调试闭环中的报文链路

> 报文采集是 T2 §5 调试闭环 `OBSERVE` 阶段的数据来源。`Observer`（T2 §5.2 的确定性 Tool）同轮同 `resource_id` 取「工程值 + 收发帧」，组装成 `RoundObservation` 喂给 `DiagnoseAgent`。本节定义这条链路的端口、数据结构与调度。

### 3.1 OBSERVE 阶段的数据组装

调试控制器每轮在 `OBSERVE` 调 `Observer.Observe`，它并行取两路数据并按 `seq` 合并：

```
每轮 OBSERVE：
    spots  = Xboard.CollectValue(resource_id)     // 工程值（解析+变比后）
    frames = SampleFrames(Xboard.GetDebug, …)      // 原始帧 + 命令 + 错误码（§3.2）
    obs    = RoundObservation{Spots: spots, Frames: frames}   // 按 seq 对齐
```

- **采集范围覆盖全表（含已收敛点）**：`OBSERVE` 始终采全表，**包括已进入收敛集的点**，用于回归校验（守 T2 §5.5「不降级」硬约束）；这与逐轮单调收缩的**调试目标集**（仅非收敛点参与 `DIAGNOSE→HYPOTHESIZE→ACT`）是两件事——"观测集 ⊋ 调试目标集"。
- **seq 单一锚点**：`debug/info` 的 `commands[].spots[].resource_id` 与 `CollectValue` 的 `Spot.resource_id` 同形（`{device}_{seq}`），直接挂到 `Observer` 既有的 `seq ↔ spot ↔ point` 绑定。
- **debug_id 复用**：调试会话 `Session.DebugID`（UUID）直接作为 `debug/info` 的 `debug_id` 入参；整个自收敛 loop 复用同一 `debug_id` 与同一会话容器。
- **帧缺失即降级**：取帧失败时 `RoundObservation.Frames` 为空，本轮 `DiagnoseAgent` 退化为「仅工程值」判定（与单次采样失败不致命一致）；不阻断闭环。
- **每轮重部署即 `/updateTemplate`**：上一轮 `ACT` 的修改经 `Deploy`（覆盖 `{board_type}.xlsx` + `/updateTemplate`）热重载后，本轮 `OBSERVE` 才采到新表数据；`Restore` 同理（重写旧 xlsx + `/updateTemplate`）。设备串行锁（系统层）照常。

### 3.2 Xboard 端口的 GetDebug

`GetDebug` 是 T10 §4.3 `Xboard` 端口的方法，由 `adapters/xboardhttp` 实现（把 `GET /api/v3/project/device/debug` 翻译成端口调用，能力包对 HTTP 无感）。端口返回结构对应 §2.3 的 `data`：

```go
// Xboard 端口方法（T10 §4.3）
GetDebug(ctx context.Context, resourceID, debugID string) (DebugLog, error)

// DebugLog 对应 debug/info 在线设备返回的 data（§2.3）。
type DebugLog struct {
    ResourceID string         `json:"resource_id"`
    DebugID    string         `json:"debug_id"`
    Version    string         `json:"version"`
    Commands   []DebugCommand `json:"commands"`
}
type DebugCommand struct {
    CommandID int           `json:"command_id"`
    ErrorCode int           `json:"error_code"`
    ErrorMsg  string        `json:"error_msg"`
    Interact  []DebugPacket `json:"interact"` // 收发帧字符串
    Spots     []DebugSpot   `json:"spots"`    // 本命令解析测点
}
type DebugPacket struct {
    Packet string `json:"packet"` // "HH:MM:SS.mmm snd|rcv: <hex>"
}
// DebugSpot 与 CollectValue 的 Spot 同锚（resource_id = {device}_{seq}）。
type DebugSpot struct {
    Name, ResourceID, Value, ID string
    Status                      int
}
```

`xboardhttp` 实现注意：`debug/info` 走 `/api/v3/project/device/debug` 路由（与采集接口不同前缀），构造请求时用对应绝对路径，外层 envelope（`{error_code,error_msg,data}`，`error_code=="00"` 成功）沿用统一解析。

### 3.3 报文解析与 seq 关联

`SampleFrames` 把 `DebugLog` 归一为「按 seq 聚合的帧」，与工程值对齐挂载：

```go
// 解析 "10:38:24.182 rcv: 01 03 14 09 12" → Frame{Dir:"rx", Hex:"01 03 14 09 12"}
type Frame struct {
    Dir   string // "tx"(snd) | "rx"(rcv)
    Hex   string // "01 03 14 09 12"
    Bytes []byte // 去空格转字节，便于 CRC / 异常码派生
    TsMs  int64  // 由 HH:MM:SS.mmm 还原
}

// FrameAgg 一条命令一轮往返的报文（按 command 聚合，经 spots 映射到 seq）。
type FrameAgg struct {
    CommandID int
    Request   *Frame // snd
    Response  *Frame // rcv
    ErrorCode int    // 透传 commands[].error_code
    ErrorMsg  string
    Seqs      []int  // 由 commands[].spots[].resource_id 解析出的读表序号
}
```

归一规则：① `interact[]` 按 `snd:`/`rcv:` 拆成 `Request`/`Response`；② `commands[].spots[].resource_id`（`{device}_{seq}`）解析出 `seq`，建立 `seq → FrameAgg` 反查，对接 `Observer` 既有 `seq` 绑定。

### 3.4 调试态保活与采样节奏（对齐 30s/100）

把 §2.2 的「布防 + 30s 超时 + 100 上限 + 读取清空」与调试闭环多轮采样节奏对齐：

| 设计点 | 规则 |
|---|---|
| 轮内节奏 | 一轮内 `SampleFrames` 以**远小于 30 秒**的周期（推荐 2–3s）轮询 `GetDebug`，每次都续命 `debugBeginTime_` 并 `swap` 取走帧；与工程值采样交错或并行 |
| 首调布防 | 每轮首次 `GetDebug` 即对设备布防（xboard 自动完成）；首轮可能需等一个采集周期帧才齐 |
| 重部署后 | 每轮 `ACT` 的 `/updateTemplate` 会**重建该 board_type 板卡并重置采集**（T1 §2.3：`DeleteBoard`+`AddBoard`）；故重部署后须**等一个采集周期**再 `OBSERVE`，且对新板卡重新布防（首调 `GetDebug` 自动完成）|
| 跨轮间隔 | 跨轮若间隔 > 30s，xboard 自动撤防，下一轮首调重新布防——可接受（每轮本就重新读取最新帧）|
| 无丢约束 | `命令数 × (轮询间隔 / 采集周期) < 100`；命令多的设备需调小轮询周期（§2.2 ④）|
| 降级 | `GetDebug` 失败/超时不致命：记录后该轮退化为「仅工程值判定」 |

### 3.5 诊断数据结构

报文随 `seq` 锚点进入 `Observer` 产物与 `DiagnoseAgent` 输入/输出，无需独立事件总线：

```go
// RoundObservation —— Observer 单轮产物（T2 §5.2）
type RoundObservation struct {
    Spots  map[string]*SpotAgg  // seq/spot → 工程值聚合
    Frames map[int]*FrameAgg    // seq → 本轮报文（§3.3）
}

// DiagnoseAgent 逐点输入：工程值 + 收发帧 + 命令错误码
type DiagnosePointInput struct {
    Seq          int
    PointName    string
    FunctionCode int
    RegisterNo   int
    Parser       string
    Multiplier   string
    Unit         string
    Samples      []string `json:"samples"`        // 工程值序列
    RequestHex   string   `json:"request_hex,omitempty"`
    ResponseHex  string   `json:"response_hex,omitempty"`
    CommandError string   `json:"command_error,omitempty"` // commands[].error_msg
}

// Verdict —— 诊断产物（debug 包权威类型，T2 §5.3）：报文字段供前端与证据链
type Verdict struct {
    // Seq/SpotID/PointName/Level/Reason/SuspectFields/Samples/Confidence/EvidenceRef/Locked …
    RequestHex   string `json:"request_hex,omitempty"`
    ResponseHex  string `json:"response_hex,omitempty"`
    ModbusExcept int    `json:"modbus_except,omitempty"` // 后端从 ResponseHex 派生
    CrcOK        bool   `json:"crc_ok"`                  // 后端对 ResponseHex 校验
}
```

> `Verdict` 的字段定义以 T2 §5.3 为权威，本节列报文相关字段以闭合数据链；`RequestHex/ResponseHex/ModbusExcept/CrcOK` 由后端确定性派生（非 LLM 产出），符合 T2 §1.3「确定性优先」。

---

## §4 喂给 DiagnoseAgent

报文增强后，`DiagnoseAgent` 的输入从「仅工程值」升级为「工程值 + 收发帧 + 命令错误码」，其 prompt 与 `Verdict` 据此扩展。可诊断结论：

| AI 诊断结论 | 依据字段 |
|---|---|
| 数值合理性（超量程、偏大 10 倍、符号异常） | `Samples`（工程值） |
| 变比 / 解析函数错误 | `Samples` + `RequestHex`/`ResponseHex`（确认寄存器地址、对照原始字节） |
| Modbus 异常码（02 非法地址 / 03 非法值） | `ResponseHex` 派生 `ModbusExcept` |
| 字节序错误 | `ResponseHex` 原始字节 vs 解析值对照 |
| CRC 校验错 / 无响应 / 超时 | `ResponseHex`（CRC）+ `CommandError`（`commands[].error_msg`） |

接入要点：

- **prompt 扩展**：`DiagnoseAgent` 的版本化 prompt 在点位字段后追加 `request_hex`/`response_hex`/`command_error`，并补充「如何据帧判定异常码/字节序/CRC」的判定说明。无帧时这些字段省略，prompt 退化为仅工程值判定。
- **Verdict 留痕**：判定结果带回 `RequestHex`/`ResponseHex`/`ModbusExcept`/`CrcOK`，既供前端「收发报文」展示，也作为后续 `HYPOTHESIZE`/`ACT`/`VERIFY` 闭环（T2 §5）与证据链的依据。
- **闭环复用**：T2 §5 的自收敛闭环 `OBSERVE → DIAGNOSE →（自动锁定正确点）→ HYPOTHESIZE → ACT(/updateTemplate 重部署) → VERIFY` 每轮 `OBSERVE` 都消费本节的 `RoundObservation`（值 + 帧，覆盖全表含收敛点），`VERIFY` 的回采即下一轮 `OBSERVE`，无需额外数据通道；loop 复用同一会话容器与 `debug_id`。

---

## §5 与桌面端的衔接（T4 / T8）

调试输出 `Session.Rounds[].Verdicts` 是 T4 H 域 UI 的数据源；接入报文后，同一份 `Verdict` 同时承载「实时值」与「收发报文」：

| T4 接口 / UI | 数据来源（`Verdict`） |
|---|---|
| H-9 实时值（右栏「实时调试」） | `Verdict.Samples` / `Level` / `Status` |
| H-10 收发报文（右栏「收发报文」） | `Verdict.RequestHex` / `ResponseHex` / `ModbusExcept` / `CrcOK` |

按 T8（gRPC Bridge）流式约定，后端把每轮 `Verdict` 经 gRPC Server Streaming 推给 Wails Bridge，Bridge 经 `runtime.EventsEmit` 推送前端：

```
控制器每轮 Verdict ──gRPC StreamRealtime/StreamFrames──► Bridge ──EventsEmit──► 前端「实时调试 / 收发报文」Tab
```

即 T8 的 `StreamRealtime`（实时值，来自 `Verdict.Samples/Level`）与 `StreamFrames`（收发报文，来自 `Verdict.RequestHex/ResponseHex`）**同源于同一份 `Verdict`**，不是两个独立数据通道。进度事件本身经 T2 §3.10 的 `ProgressSink` 外推，由系统层适配为 gRPC 流。

---

## §6 与其他文档的对齐

| 文档 | 对齐点 |
|---|---|
| **T1 §1.6** | xboard 集成接口在 `Update`/`Status`/`CollectValue` 基础上含 `GetDebug`（`/api/v3/project/device/debug`），作为调试报文数据源 |
| **T2 §5** | 报文是 `OBSERVE` 阶段数据来源；`Observer` 产 `RoundObservation{Spots,Frames}`，`DiagnoseAgent` 消费值+帧，`Verdict` 含 `RequestHex/ResponseHex/ModbusExcept/CrcOK`（§5.3 为字段权威） |
| **T10 §4.3 / §6** | `Xboard` 端口含 `GetDebug`，由 `adapters/xboardhttp` 实现；`Observer` 的取帧逻辑（`SampleFrames`/`FrameAgg`）落在 `capability/debugging` |
| **T4 H-9 / H-10** | H-9 实时值来自 `Verdict.Samples/Level`，H-10 收发报文来自 `Verdict.RequestHex/ResponseHex`；hex 源自 `debug/info` 的 `interact[].packet` |
| **T8** | `StreamRealtime` / `StreamFrames` 同源于同一份 `Verdict`，桌面↔服务端经 gRPC 推送 |

> 报文链路对外只依赖 xboard `debug/info`（HTTP，服务端↔外部服务）这一条数据源；其余命名（`Observer`/`RoundObservation`/`DiagnoseAgent`/`Xboard` 端口/`Verdict`）均与 T2、T10 一致。
