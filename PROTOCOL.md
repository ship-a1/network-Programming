# 应用层协议与接口契约（PROTOCOL.md）

> **唯一协议源。成员 C 负责维护，A / B 只读。**
> 冻结后任何字段变更必须三人同步确认并递增 `ver`；禁止某一方单独改自己那侧。
> 当前版本 **`ver = 1`**。
> 上游依据：`README.md`（技术栈与工程约定）、`计网分工.docx`（任务归属）、《计算机网络课程实验选题说明》（TCP/UDP 与实验指标要求）。

---

## 1. 三条链路与帧格式

| 链路 | 端点 | 消息边界 | 参与者 |
|---|---|---|---|
| **WS** | `ws://<host>:8080/ws?role=<role>&room=<room>` | 一条 WebSocket text frame = 一个 JSON 对象（WS 自带边界，**无长度前缀**） | B ↔ C |
| **TCP** | `127.0.0.1:9000` | **4 字节大端长度前缀 + UTF-8 JSON**（见 1.2） | C ↔ A |
| **UDP** | `127.0.0.1:9001` | 一个数据报 = 一个 JSON 对象（无重传、无确认） | C 网关 → C 的 udp_probe（仅对照实验） |

### 1.1 WS 侧约定

- 连接时 URL 必带 `role`，取值 `register | doctor | display | net`；`doctor` 页另带 `room`（如 `room=3`）。
- 浏览器**不发 HELLO**：C 已由 URL 获知角色。C 为该 WS 建立**一条独立 TCP 连接**到 A 并代发 `HELLO`（见第 7 节时序 1）。
- 连接模型：**1 条 WS ↔ 1 条 TCP**，连接映射由 C 维护。A 因此看到的是多个真实客户端连接。
- C 对端到端业务消息**只透传**：仅在 `req_id` 缺失时补齐并登记，不增删任何业务字段。

### 1.2 TCP 侧帧格式（拆包 / 粘包契约）

```
帧 = [ 4 字节 N ][ N 字节 UTF-8 JSON ]
     N = struct.pack(">I", N)   # 大端无符号 32 位
```

- `N` 为 **JSON 的 UTF-8 字节数**，合法范围 `1 ≤ N ≤ 65536`；`N = 0` 或 `N > 65536` 视为协议错误，回 `ERROR 1001` 并断开连接。
- 一帧一条消息，**禁止按换行符切分**（JSON 内可能含中文与空格，不得依赖 `\n`）。
- 接收方必须维护字节缓冲区：不足 4 字节继续收 → 解析出 N → 不足 N 字节继续收 → 切出一帧解码 → 剩余字节留在缓冲区并**循环处理**（一次 `recv` 可能含多帧）。
- 帧编解码由 **A、C 各实现一份**（约 10 行）；C 额外提供边界用例（半包 / 粘包 / 超长帧）供双方自测。

**帧示例**（可直接用于 A 的自测）：

```json
{"ver":1,"type":"REGISTER","req_id":"reg-1","ts":1732200000.0,"role":"register","data":{"dept":"A"}}
```

该 JSON 共 **100 字节**，完整帧前 4 字节为 `00 00 00 64`，其后紧跟上述 100 字节。

### 1.3 UDP 侧约定

- 网关每收到一条 `STATE_UPDATE`，除走 WS 推给页面外，**复制一份** `UDP_STATE` 发往 `127.0.0.1:9001`；probe 接收并统计。
- `UDP_STATE` 与 `STATE_UPDATE` 的 `data` 同构，额外带 `sent_ts`。
- **A 不实现 UDP，B 不感知 UDP**；UDP 不承载任何关键操作，只用于生成 TCP/UDP 接收率与时延对照数据（选题说明要求）。

---

## 2. 统一 JSON 信封（所有消息）

```json
{
  "ver": 1,
  "type": "CALL_NEXT",
  "req_id": "doc-7",
  "ts": 1732200000.123,
  "role": "doctor",
  "seq": 42,
  "data": {"room": "3"},
  "code": 0,
  "msg": "OK"
}
```

| 字段 | 类型 | 必填 | 填写方 | 说明 |
|---|---|---|---|---|
| `ver` | int | ✓ | 发起方 | 协议版本，恒为 `1`；收到不支持的版本回 `ERROR 1004` |
| `type` | str | ✓ | 发起方 | 消息类型，大写下划线，取值见第 3 节；未知类型回 `ERROR 1002` |
| `req_id` | str | ✓ | **B 生成**，C 校验补齐 | 端到端唯一标识，格式 `<role缩写>-<自增>`（如 `reg-1`、`doc-7`）。**响应 / ACK / ERROR 必须原样带回**；仅 WS 侧的 `CLIENT_RENDER` / `METRICS` / `MOCK_STATE` 填 `""` |
| `ts` | float | ✓ | 发起方 | `time.time()`，Unix 秒（float），发送时刻；用于 RTT 与更新延迟计算 |
| `role` | str | ✓ | 发起方 / C | 端到端业务消息**透传原始浏览器角色**；C↔A 控制消息填 `gateway`；A 下发的消息填 `server` |
| `data` | object | ✓ | 发起方 | 业务负载，无内容时 `{}`；字段见第 3 节 |
| `seq` | int | 条件 | **仅 A** | 服务端状态版本，单调递增；仅 `ACK` / `SNAPSHOT` / `STATE_UPDATE` / `HELLO_ACK` / `PONG` 携带 |
| `code` | int | 条件 | **仅 A** | 仅响应类消息（`ACK` / `ERROR` / `HELLO_ACK` / `SNAPSHOT`），`0` = 成功，非 0 见第 5 节 |
| `msg` | str | 条件 | A / C | 人类可读的结果或错误文本，成功填 `"OK"` |

**JSON 编码**：`ensure_ascii=False`、UTF-8、紧凑分隔符 `(",", ":")`，字段顺序不限。

---

## 3. 消息类型总表

链路列：`WS+TCP` = 端到端（B 发 → C 透传 → A 处理 → 回传）；`TCP` = 仅 C↔A；`WS` = 仅 B↔C；`UDP` = 仅 C 内部。

### 3.1 业务请求（端到端，B → C → A）

| `type` | 链路 | 发起 `role` | `data` | A 的响应 |
|---|---|---|---|---|
| `REGISTER` | WS+TCP | `register` | `{dept:"A", name:"测试001"}`（虚构姓名，可空） | `ACK`，`data` 带回 `{ticket_no, pos}` |
| `QUERY` | WS+TCP | 任意 | `{}` | `SNAPSHOT`（全量队列） |
| `CALL_NEXT` | WS+TCP | `doctor` | `{room:"3"}` | `ACK`，`data` 带回 `{ticket_no, room}`；队列空回 `2001` |
| `RECALL` | WS+TCP | `doctor` | `{ticket_no:"A003"}` | `ACK`（重叫当前号，不改变队列顺序，`seq` +1） |
| `SKIP` | WS+TCP | `doctor` | `{ticket_no}` | `ACK`（过号，状态置 `skipped`） |
| `FINISH` | WS+TCP | `doctor` | `{ticket_no}` | `ACK`（完成就诊，状态置 `done`） |

> 六个业务操作覆盖分工文档中 A 的全部职责（取号、查询、叫号、重叫、过号、完成）。

### 3.2 服务端下发（A → C → B）

| `type` | 链路 | 方向 | `data` | C 的处理 |
|---|---|---|---|---|
| `ACK` | TCP → WS | A → 原请求方 | `{code, msg, seq, ...业务结果}` | **按 `req_id` 路由回原 WS 连接**，同时登记 RTT |
| `SNAPSHOT` | TCP → WS | A → 单个连接 | `{code, msg, seq, current, waiting, rooms, ts_server}` | 转给对应 WS；页面**全量重渲染** |
| `STATE_UPDATE` | TCP → WS（广播） | A → **所有**连接 | `{seq, current, waiting, stats, ts_server}` | 推给**所有** WS；`display` 渲染、`net` 记日志，其余页面可只更新自身状态 |
| `ERROR` | TCP → WS | A → 原请求方 | `{code, msg, req_id}` | 按 `req_id` 转回原 WS；`net.html` 记入 `msg_log` |
| `HELLO_ACK` | TCP | A → C | `{code, msg, seq, server_time}` | 标记该 TCP 连接 ready，开始转发 |

### 3.3 连接与可靠性（仅 TCP 侧，B 不参与）

| `type` | 链路 | 方向 | `data` | 说明 |
|---|---|---|---|---|
| `HELLO` | TCP | C → A | `{role, room, client_id}` | 每条新 TCP 连接的**首帧**；A 登记角色与诊室；未 HELLO 就发业务消息回 `3001` |
| `PING` | TCP | C → A | `{ts}` | 每 5 s 一次，A 原样回 `ts` |
| `PONG` | TCP | A → C | `{ts, seq}` | 用于 RTT 与半开连接检测 |
| `RESYNC` | TCP | C → A | `{last_seq}` | 重连后请求状态恢复；A 比对 `seq`，落后则回 `SNAPSHOT`，否则回 `ACK{code:0, seq}` |

### 3.4 仅 WS 侧（B ↔ C，A 不参与）

| `type` | 链路 | 方向 | `data` | 说明 |
|---|---|---|---|---|
| `CLIENT_RENDER` | WS | B(display) → C | `{seq, rendered_ts}` | `display.html` 每次收到 `STATE_UPDATE` **并完成 DOM 更新后**立即上报；C 据此算候诊屏更新延迟（见 8 节） |
| `METRICS` | WS | C → B(net) | 指标精简对象 | 每 1 s 广播一次，供 `net.html` 实时展示消息类型 / RTT / 连接状态 / TCP-UDP 对照 |
| `MOCK_STATE` | WS | C → B | 与 `SNAPSHOT` / `STATE_UPDATE` 同构 | **仅 `--mock` 模式**，供 B 脱离 A、C 真实链路独立开发页面 |

### 3.5 UDP 对照（仅 C 内部）

| `type` | 链路 | 方向 | `data` | 说明 |
|---|---|---|---|---|
| `UDP_STATE` | UDP | C 网关 → C probe | `STATE_UPDATE.data` + `sent_ts` | 统计发送/接收数与延迟，产出 `udp.recv_rate`；丢失不重传、不补发 |

---

## 4. 队列状态模型与 `seq` 规则（A 为唯一权威）

### 4.1 状态对象

```json
{
  "seq": 42,
  "current": {"ticket_no": "A003", "room": "3", "called_ts": 1732200123.456},
  "waiting": [
    {"ticket_no": "A004", "dept": "A", "created_ts": 1732200120.1, "state": "waiting"},
    {"ticket_no": "A005", "dept": "A", "created_ts": 1732200121.9, "state": "waiting"}
  ],
  "rooms": {"3": {"ticket_no": "A003"}},
  "stats": {"total": 5, "waiting": 2, "done": 2, "skipped": 1},
  "ts_server": 1732200123.456
}
```

- `ticket_no`：单科室原型，格式 `A` + 三位序号（`A001` 起，A 端递增，不复用）。
- `state` 枚举：`waiting` / `calling` / `skipped` / `done`。
- `current` 为 `null` 表示尚未叫号。
- `waiting` 为**有序数组**，顺序即队列顺序；B 直接按数组顺序渲染，不得自行排序或推算。

### 4.2 `seq` 规则（一致性的核心）

1. `seq` 由 **A 独占分配**，初值 `0`；每次状态变更（六个业务操作任一成功）`+1`。
2. 随 `ACK` / `SNAPSHOT` / `STATE_UPDATE` / `HELLO_ACK` / `PONG` 下发。
3. **B 与 C 只接受 `seq` 严格大于本地已见最大值的状态帧**，旧帧直接丢弃，保证乱序/重连时不回退。
4. A 用 `threading.Lock` 串行化「状态变更 → seq+1 → 广播」，保证并发取号不产生重复票号。
5. 重连时 C 带 `last_seq` 发 `RESYNC`；A 若发现其落后，回 `SNAPSHOT` 全量恢复。

---

## 5. 错误码表（C 定义，A 实现，B 原样展示）

| `code` | 含义 | 处理建议 |
|---|---|---|
| `0` | OK | — |
| `1001` | 消息解析失败 / JSON 非法 / 帧长度非法 | 记录日志；帧长度非法时断开连接 |
| `1002` | 未知 `type` | 回 `ERROR`，不断连 |
| `1003` | 缺少必填字段 | 回 `ERROR`，指出缺失字段 |
| `1004` | 协议版本不支持（`ver != 1`） | 回 `ERROR`，提示统一版本 |
| `2001` | 队列为空（`CALL_NEXT` 时） | 医生页提示「暂无候诊」 |
| `2002` | 票号不存在 | 医生页提示 |
| `2003` | 票号当前状态不允许该操作 | 医生页提示 |
| `2004` | 重复请求（`req_id` 已处理） | **A 不重复执行业务**，直接回上一次结果（`code:0`），用于超时重试去重 |
| `3001` | 未 `HELLO` / 连接未登记 | C 重新建连并发 `HELLO` |
| `4001` | 服务端内部错误 | 记入日志，`net.html` 标红 |

---

## 6. 可靠性参数（写死，三方不得各定各的）

| 项 | 值 | 责任方 |
|---|---|---|
| 请求 `ACK` 超时 | **2 s**；超时用**同一 `req_id`** 重发，最多 2 次（共 3 次） | C |
| 重复请求去重 | A 缓存已处理的 `req_id` → 结果，命中回 `2004` 或原结果，不重复执行业务 | A |
| 心跳 | C 每 **5 s** 发 `PING`；连续 **3 次**无 `PONG` 判定断线 | C 发 / A 回 |
| 重连退避 | 1 s → 2 s → 4 s，上限 8 s | C |
| 重连恢复 | `HELLO` → `RESYNC{last_seq}` → 落后则收 `SNAPSHOT` → 页面全量重渲染 | C / A / B |
| 连接握手超时 | 3 s（C 连 A） | C |
| 单帧长度上限 | 65536 B | A / C |
| 时间戳 | 统一 `time.time()`（Unix 秒 float） | 全体 |
| 指标广播间隔 | 1 s（`METRICS`） | C |

---

## 7. 关键时序

1. **连接建立**：打开页面 → `common.js` 连 `ws://...?role=...&room=...` → C 为该 WS 建一条 TCP 到 9000 → 发 `HELLO{role, room, client_id}` → A 回 `HELLO_ACK{seq}` → C 标记 ready，开始转发。
2. **患者取号**：`REGISTER` → C 登记 `req_id` 与 `send_ts` 并透传 → A 加锁生成票号、入队、`seq+1` → 回 `ACK`（同 `req_id`，含 `ticket_no/pos`）→ **同时**向所有连接广播 `STATE_UPDATE` → C 按 `req_id` 把 `ACK` 回原 WS，把 `STATE_UPDATE` 推给所有 WS。
3. **医生叫号**：`CALL_NEXT{room}` → A 取队首、设 `current`、`seq+1` → `ACK` + 广播；`display` 渲染最新当前号与后续队列，`doctor` 页更新按钮状态。
4. **大屏实时更新**：`display` 收到 `STATE_UPDATE` → 更新 DOM → 立即发 `CLIENT_RENDER{seq, rendered_ts}`；C 计算屏更新延迟（第 8 节）。全程无需手动刷新。
5. **多用户并发**：每个 tab 一条 WS↔TCP；A 用 `Lock` 串行化状态变更，`seq` 全局单调；广播发给全部连接，所有 `display` 页看到同一份队列。
6. **超时重试**：C 2 s 未收 `ACK` → 同 `req_id` 重发 → A 命中去重缓存，回同一结果，不产生第二个票号。
7. **断线重连**：关闭/刷新页面 → 对应 WS 与 TCP 断开 → 重新连接 → C 新建 TCP、`HELLO` → `RESYNC{last_seq}` → A 判定落后则 `SNAPSHOT` → C 转给页面全量重渲染；C 记录 `reconnect_ms`。
8. **网络信息展示**：C 每 1 s 广播 `METRICS` → `net.html` 展示消息类型流水、RTT、连接状态与 TCP/UDP 接收率对照。

---

## 8. 指标埋点契约（补齐分工文档「待定」）

### 8.1 谁负责什么

| 指标 | 计算方式 | 数据来源 |
|---|---|---|
| `active_ws` / `active_tcp` | C 直接计数活跃连接 | C |
| `rtt_ms`（请求响应时间） | `ACK.recv_ts − 请求.send_ts`，按 `req_id` 配对 | C 登记 |
| `screen_lag_ms`（候诊屏更新延迟） | `CLIENT_RENDER.rendered_ts − STATE_UPDATE.ts_server`，按 `seq` 配对 | **B 上报 rendered_ts** + A 的 `ts_server` |
| `reconnect_ms` | `SNAPSHOT 收到时刻 − 断线时刻` | C |
| `tcp_ack_rate` | 去重后收到的 `ACK` 数 / 发出的业务请求数 | C |
| `udp.recv_rate` / `udp.avg_delay_ms` | probe 收到数 / 网关发出数；`recv_ts − sent_ts` | C 的 UDP 对照通道 |
| `msg_log` | 最近 200 条 `{ts, direction, type, req_id, code, seq}` | C |

> **时钟说明**：本机演示（多浏览器窗口）三方同机，`time.time()` 可直接相减。若改用多台局域网设备，`screen_lag_ms` 改为 C 侧「WS 转发时刻 − `ts_server`」，并在报告中注明口径变化。

### 8.2 输出

- `metrics.json`（C 运行时生成于项目根，结构如下）
- `GET /api/metrics` 返回同一对象
- `METRICS` 消息 `data` 为其精简版（含 `active_ws`、最近 `rtt_ms`、`msg_log` 尾部、`udp` 摘要），1 s 一次推给 `net.html`

```json
{
  "generated_at": 1732200000.0,
  "active_ws": 5,
  "active_tcp": 5,
  "rtt_ms": {"count": 120, "avg": 14.8, "min": 3.1, "max": 61.2},
  "screen_lag_ms": {"count": 45, "avg": 21.3, "min": 8.0, "max": 55.7},
  "reconnect_ms": {"count": 3, "avg": 1120.0},
  "tcp_ack_rate": 1.0,
  "udp": {"sent": 45, "recv": 44, "recv_rate": 0.978, "avg_delay_ms": 2.4},
  "msg_log": [
    {"ts": 1732200000.12, "direction": "out", "type": "REGISTER", "req_id": "reg-1", "code": null, "seq": null}
  ]
}
```

> ⚠️ 上表数值**仅示意 JSON 结构**。分工文档与选题说明都强调：`15 ms` / `20 ms` 一类示例值不得直接写入实验报告，必须使用程序实测数据。

---

## 9. 演示六场景 → 消息映射

| 场景 | 操作 | 涉及消息 | 现场观察点 |
|---|---|---|---|
| 1 患者取号 | 挂号页点击「取号」 | `REGISTER` → `ACK` → `STATE_UPDATE` | 自动生成票号与队列位置 |
| 2 医生叫号 | 医生页点击「叫号」 | `CALL_NEXT` → `ACK` → `STATE_UPDATE` | 当前就诊号自动更新 |
| 3 大屏实时更新 | 无操作 | `STATE_UPDATE` → `CLIENT_RENDER` | 不手动刷新即同步 |
| 4 多用户同时访问 | 同时开多个浏览器窗口 | 多条 WS↔TCP、`seq` 广播 | 共享同一份队列；A 端连接数 = tab 数 |
| 5 断线重连 | 关闭/刷新一个页面再连 | `HELLO` → `RESYNC` → `SNAPSHOT` | `seq` 与 `waiting` 与重连前一致 |
| 6 网络通信信息展示 | 打开 `net.html` | `METRICS` + `msg_log` + `udp` 统计 | 消息类型、RTT、连接状态、TCP/UDP 接收率对照 |

> 场景 6 的 TCP/UDP 对照数据来自第 1.3 节的 C 内部通道；关键叫号操作始终走 TCP + ACK，符合选题说明「UDP 用于非关键状态更新实验」。

---

## 10. 附录：协议自测判定标准（联调前各自验证）

| 用例 | 构造方式 | 期望结果 |
|---|---|---|
| 半包 | 把一帧拆成两次 `send` | 接收方缓冲区等待，凑齐 N 字节才解码出一条消息 |
| 粘包 | 一次 `send` 连续两帧 | 循环切分，解出两条独立消息，顺序不变 |
| 超长帧 | 前缀 `N = 70000` | 回 `ERROR 1001` 并断开该连接，不影响其他连接 |
| 非法 JSON | 帧内容为非 JSON 文本 | 回 `ERROR 1001`，连接保持 |
| 重复 `req_id` | 同一 `req_id` 发两次 `REGISTER` | 只生成一个票号，第二次回同一结果（`2004` 或原 `code:0`） |
| 并发取号 | 5 条连接同时 `REGISTER` | 票号连续且不重复，`seq` 递增无跳变，广播 5 条 `STATE_UPDATE` |
| 旧 `seq` | 人为推送 `seq` 更小的 `STATE_UPDATE` | B / C 丢弃，不渲染、不回退 |
| 未 HELLO | 建连后直接发业务消息 | 回 `ERROR 3001` |
| 重连恢复 | 断线 → 重连 → `RESYNC{last_seq}` | 收到 `SNAPSHOT`，`waiting` 与断线前一致 |
