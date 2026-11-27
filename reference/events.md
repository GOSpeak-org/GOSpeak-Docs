# 信令事件

除 HTTP 接口外，前端还通过一条 WebSocket 长连接接收实时事件，包括成员加入、发言状态与新消息等。

## 连接

```
ws(s)://你的域名/ws
```

握手鉴权由 HttpOnly access cookie 自动携带；子协议只传协议名 `gospeak`，不携带 token。

事件协议的单一事实源在后端 `internal/protocol`（Catalog 声明事件名 / 方向 / ACK / payload 类型），由生成器产出前端 `src/protocol/generated/`；payload 字段一律 snake_case。

## 事件表

方向说明：`C→S` 为客户端发给服务端，`S→C` 为服务端推送给客户端。**ACK** 列 ✓ 表示该上行事件走「emit → ack」模式：handler 返回纯 data 作为 ack 载荷，业务错误时返回 error ACK（前端 Promise reject）。

### 房间

| 事件 | 方向 | ACK | 说明 |
|------|------|:---:|------|
| `room:create` | C→S | | 创建房间 |
| `room:created` | S→C | | 房间创建完成 |
| `room:join` | C→S | ✓ | 加入房间（成功 ack 返回 `{room, identity}`；失败 ack 带 `error` 字段；成员列表由 `room:join:sfu` 的 ack `members` 与 `room:updated` / `room:list:result` 补齐，`member:joined` 仅通告新加入的单个成员） |
| `room:leave` | C→S | ✓ | 离开房间 |
| `room:left` | S→C | | 已离开 |
| `room:list` | C→S | | 请求房间列表 |
| `room:list:result` | S→C | | 房间列表结果 |
| `room:updated` | S→C | | 房间信息变更 |
| `room:kick` | C→S | | 移出成员（需 `signal:kick` 权限） |
| `room:kicked` | S→C | | 当前连接被移出房间 |
| `room:join:sfu` | C→S | ✓ | SFU 媒体连接确认后通知服务端，写入成员并广播 `member:joined` |
| `room:active-speakers` | S→C | | 当前正在发言的成员（批量） |

### 成员

| 事件 | 方向 | ACK | 说明 |
|------|------|:---:|------|
| `member:joined` | S→C | | 成员加入 |
| `member:left` | S→C | | 成员离开 |
| `member:updated` | S→C | | 成员信息变更 |
| `member:speaking` | C→S | | 上报本端发言状态（协议与服务端 handler 已就位，但当前前端不发送；发言高亮实际来自 SFU 原生 `onActiveSpeakers` 与 `room:active-speakers` 双源，勿依赖本事件） |
| `member:mic-state` | C→S | | 上报本端麦克风开关状态 |
| `member:muted` | S→C | | 成员麦克风被服务端静音 |
| `member:unmuted` | S→C | | 成员麦克风恢复 |

### 消息

| 事件 | 方向 | ACK | 说明 |
|------|------|:---:|------|
| `message:send` | C→S | ✓ | 发送消息（ack 无 data；落库后的 DTO 由 `message:created` 广播） |
| `message:edit` | C→S | ✓ | 编辑消息 |
| `message:delete` | C→S | ✓ | 删除消息 |
| `message:react` | C→S | ✓ | 添加表情反应 |
| `message:unreact` | C→S | ✓ | 移除表情反应 |
| `message:created` | S→C | | 新消息广播 |
| `message:updated` | S→C | | 消息已编辑 |
| `message:deleted` | S→C | | 消息已删除 |
| `message:reaction` | S→C | | 表情反应变更 |

### 私聊

| 事件 | 方向 | ACK | 说明 |
|------|------|:---:|------|
| `private:send` | C→S | ✓ | 发送私聊消息 |
| `private:new` | S→C | | 收到私聊消息 |

### 其他

| 事件 | 方向 | ACK | 说明 |
|------|------|:---:|------|
| `typing` | 双向 | | 正在输入 |
| `user:muted` | S→C | | 当前用户被禁言（MuteInfo：`user_uuid`、禁言原因/到期时间、`enforcement` 媒体强制级别） |
| `user:unmuted` | S→C | | 当前用户禁言被解除 |
| `announcement:new` | S→C | | 系统公告推送 |
| `sfu:provider-changed` | S→C | | livekit 配置更新导致媒体会话需重建时广播（payload 含 `provider`）；前端收到后重连媒体会话，不再表示 provider 切换 |

## 进房完整流程

```
前端                          后端                    LiveKit
 │                             │                        │
 ├─ POST /signal/token ───────►│                        │
 │◄──── 令牌 + 地址 ───────────┤                        │
 │                             │                        │
 ├─ ws 连接 ──────────────────►│                        │
 │  (HttpOnly cookie 鉴权)     │                        │
 │                             │                        │
 ├─ room:join ────────────────►│                        │
 │◄──── ack（room + identity）─┤                        │
 │                             │                        │
 ├─ 用令牌连 LiveKit ──────────────────────────────────►│
 │◄───────────────────── 音频流 ───────────────────────┤
 │◄──── onActiveSpeakers（原生发言态，写入前端高亮） ───┤
 │                             │                        │
 ├─ room:join:sfu ────────────►│                        │
 │◄──── ack（ok + room + identity + members）──────────┤
 │◄──── member:joined × N ─────┤                        │
 │◄──── room:active-speakers ──┤  （加入回放，写入高亮）│
```

关键点：**信令和媒体是两条独立的连接**。信令负责状态同步，媒体负责音频传输。

## 信令与媒体分离的原因

- 信令要求可靠（不能丢），走 TCP
- 媒体要求低延迟（能容忍少量丢包），走 UDP

两者混用会互相影响：音频卡顿不应导致状态同步延迟。

## 断线重连

前端会自动重连，重连后需要重新执行 `room:join` 以补齐状态。

重连期间状态可能不一致。重连时前端只重发 `room:join`（媒体会话不重建，也不重发 `room:join:sfu`），成员状态以 `onConnected` 拉取的 `room:list:result` 与随后的 `room:updated` 为准。`room:join:sfu` ack 的 `members` 字段只在首次建立媒体会话时用于即时补齐当前房间成员。

## 常见问题

**界面持续显示「已断开」**
反向代理未放行 WebSocket 升级（`Upgrade` / `Connection` 头）。见 [反代与 HTTPS](/deploy/reverse-proxy)。

**连接建立但收不到事件**
WebSocket 握手靠 HttpOnly cookie 鉴权，不在 URL/子协议里带 token。若 cookie 中的 access token 过期，HTTP 层会在下次请求时经 refresh_token 续期；WS 重连时前端也会重新调用 getWSEndpoint 解析端点。若事件仍不到达，先确认令牌状态与连接是否已断开重连。

**消息发出后无响应**
上行消息事件走 emit → ack 模式：ack Promise 超时（10s）或 reject 即发送失败，需检查连接状态或错误内容。

**成员列表不更新**
未收到 `member:joined` / `member:left`。通常为连接已断开但界面未感知，刷新页面可恢复。

## 下一步

- [API 参考](/reference/api) — HTTP 接口
- [错误码](/reference/errors) — 错误含义
