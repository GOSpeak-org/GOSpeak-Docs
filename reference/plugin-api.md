# 插件 API 参考

本页面向插件作者，描述插件进程与宿主之间的 gRPC 接口与 JS SDK 用法。**协议单一事实源是** [`server/plugin/proto/plugin.proto`](https://github.com/GOSpeak-org/GOSpeak-Server/blob/release/server/plugin/proto/plugin.proto)；页面只是转述，接口以 proto 为准。

> [!WARNING]
> **拉取即信任**：插件注册后即拥有宿主全部能力，没有 token、能力点、运行期权限边界或沙箱。本页所有读写接口都以系统身份委托宿主 service 执行，写操作会产生 `ActorKind=plugin` 审计。请只开发/安装可信插件。

## 通信模型

- 宿主为插件分配 **127.0.0.1 上的动态端口**，通过一次性凭据文件（`0600`）注入，环境变量名为 `GOSPEAK_PLUGIN_CREDENTIAL`。
- 插件与宿主之间走 **gRPC over 本地 TCP 回环**，不对外监听；凭据由宿主注入的一次性文件传递，插件不需要任何 token 或权限配置。
- 插件侧建议直接使用 JS SDK；SDK 内部读取凭据文件、加载 proto、建立 channel，插件作者无需接触这些细节。
- 跨语言插件可基于同一份 proto 自行生成客户端 / 服务端代码。

## gRPC 服务清单

### RegistrationService — 注册与保活

| RPC | 请求字段 | 响应字段 | 说明 |
|-----|----------|----------|------|
| `Register` | `plugin_id`、`name`、`version`、`hook_types[]`、`pages[]`、`hook_endpoint` | `api_endpoint`、`confirmed_pages[]` | 注册插件；`pages[]` 含 `route_path` / `mode` / `entry_url`。路由必须以 `/plugins/<plugin_id>/` 为前缀，经确认后返回 `confirmed_pages` |
| `Heartbeat` | `plugin_id` | 空 | 保活；宿主按心跳失败次数告警并重启 / 停用插件 |

`Register` 返回的 `api_endpoint` 即宿主 `PluginAPIService` 的回环地址；`hook_endpoint` 是插件自己监听、供宿主推送 Hook 的地址。

### PluginAPIService — 宿主能力读写

| RPC | 请求字段 | 响应字段 | 读/写 |
|-----|----------|----------|-------|
| `GetUser` | `user_uuid` | `user_uuid`、`username`、`display_name` | 读 |
| `ListRooms` | `domain_uuid` | `rooms[]`（`room_uuid`、`name`） | 读 |
| `MuteUser` | `user_uuid`、`room_uuid`、`reason`、`request_id` | `ok` | 写 |
| `CreateRoom` | `domain_uuid`、`name`、`reason`、`request_id` | `room_uuid` | 写 |
| `GetConfig` | `plugin_id` | `config_json` | 读 |
| `KVSave` | `plugin_id`、`key`、`value_json`、`reason`、`request_id` | 空 | 写 |
| `KVLoad` | `plugin_id`、`key` | `value_json`、`found` | 读 |
| `ReportError` | `plugin_id`、`message`、`details_json`、`request_id` | 空 | 上报 |

- **字段命名一律 snake_case，身份字段一律 `*_uuid`**（`user_uuid` / `room_uuid` / `domain_uuid` / `plugin_id`），不存在数字用户 ID 字段。
- 写操作（`MuteUser` / `CreateRoom` / `KVSave`）必须传 `request_id` 做幂等；宿主按 `(plugin_id, request_id)` 去重，重复请求返回与首次一致的成功结果。SDK 会自动生成 UUID。
- `MuteUser` / `CreateRoom` 由宿主以系统保留用户身份执行（该用户不可登录），不经过请求用户的权限判断。
- 插件**不直接访问主数据库**；配置经 `GetConfig` 读取，结构化数据经 `KVSave` / `KVLoad` 存取。

### PluginHookService — 事件推送

| RPC | 请求字段 | 响应字段 | 说明 |
|-----|----------|----------|------|
| `HandleEvent` | `plugin_id`、`event_type`、`payload_json`、`request_id` | 空 | 宿主把裁剪后的内部事件推送给插件；敏感事件不会下发 |

插件在 `Register` 时通过 `hook_types` 声明要订阅的事件；宿主持续失败时会停用对应插件。

### RuntimeService — 运行时版本管理

由 runtime 插件实现，宿主调用；管理员在管理页面触发。

| RPC | 请求字段 | 响应字段 | 说明 |
|-----|----------|----------|------|
| `Install` | `kind`、`version`、`mirror`、`reason`、`request_id` | `installed`、`unsupported` | 下载并校验指定版本；`kind=python` 返回 `unsupported=true`（预留） |
| `List` | `kind` | `runtimes[]`（`kind`、`version`、`active`） | 列出已安装版本；不传 `kind` 列出全部 |
| `Use` | `kind`、`version`、`reason`、`request_id` | `ok`、`unsupported` | 切换当前软链；Node 切换会触发插件重启 |

runtime **不会在启动时自动下载** 任何版本。

## JS SDK：createPlugin

安装 workspace 包 `@gospeak/plugin-sdk` 后，一个最小插件如下：

```js
import { createPlugin } from "@gospeak/plugin-sdk"

const plugin = await createPlugin({
  pluginId: "com.example.audit",
  name: "示例插件",
  version: "1.0.0",
  hookTypes: ["message.created"],
  pages: [],
  onEvent(event, payload) {
    console.log("收到 Hook:", event, payload)
  },
})

// 读取宿主 API
const user = await plugin.api.getUser("<user_uuid>")
const rooms = await plugin.api.listRooms("<domain_uuid>")

// 写操作：requestId 由 SDK 自动生成；也可显式传入以保证跨重启幂等
await plugin.api.muteUser({
  userUuid: user.userUuid,
  roomUuid: rooms[0].roomUuid,
  reason: "plugin policy",
})
const roomUuid = await plugin.api.createRoom({
  domainUuid: "<domain_uuid>",
  name: "插件创建的房间",
  reason: "plugin policy",
})

// 配置与 KV
console.log(await plugin.config.get())
await plugin.kv.save("last_room", { roomUuid })
console.log(await plugin.kv.load("last_room"))

// 额外订阅某个事件（onEvent 为兜底回调）
plugin.on("message.created", (payload) => {
  void plugin.reportError("收到消息事件", payload)
})

// 进程退出前关闭连接
await plugin.close()
```

SDK 返回对象的能力：

| 成员 | 说明 |
|------|------|
| `api.getUser(userUuid)` | 读用户，返回 `{ userUuid, username, displayName }` |
| `api.listRooms(domainUuid)` | 读房间列表，返回 `{ roomUuid, name }[]` |
| `api.muteUser({ userUuid, roomUuid, reason, requestId? })` | 禁言用户，返回 `boolean` |
| `api.createRoom({ domainUuid, name, reason, requestId? })` | 创建房间，返回 `roomUuid` |
| `on(event, handler)` | 追加订阅 Hook 事件 |
| `config.get()` | 读取管理页面配置的 JSON |
| `kv.save(key, value, reason?)` / `kv.load(key)` | 插件 KV 读写；`load` 未命中返回 `undefined` |
| `reportError(message, details?)` | 结构化错误上报（写日志） |
| `close()` | 关闭 Hook 服务与 API 连接 |

SDK 内置 proto 副本由 GOSpeak-Client 的 [`packages/plugin-sdk/scripts/copy-proto.mjs`](https://github.com/GOSpeak-org/GOSpeak-Client/blob/release/packages/plugin-sdk/scripts/copy-proto.mjs) 从权威 proto 同步，**不要手改副本**。

## 错误码映射

插件 API 返回 gRPC 标准状态码（详见 proto 与 [`server/internal/plugin/api.go`](https://github.com/GOSpeak-org/GOSpeak-Server/blob/release/server/internal/plugin/api.go)）：

| gRPC code | 触发场景 |
|-----------|----------|
| `INVALID_ARGUMENT` | 缺少必填字段（如 `user_uuid` / `domain_uuid` / `plugin_id` / `key`） |
| `NOT_FOUND` | `GetUser` 找不到用户 |
| `INTERNAL` | 宿主 service / 存储调用失败、幂等占位失败 |
| `UNAVAILABLE` | 宿主依赖未初始化 |

这些 **gRPC 码与平台 HTTP 的业务错误码不是同一套**。平台 HTTP 接口的业务码（如 `1001` 未登录、`2001` 参数错误、`3001` 资源不存在、`5001` 内部错误等）以 [`server/internal/pkg/errors.go`](https://github.com/GOSpeak-org/GOSpeak-Server/blob/release/server/internal/pkg/errors.go) 为唯一事实源，完整表见 [错误码](/reference/errors)。管理 API（`/api/v1/plugin/*`）返回统一 JSON 外壳 `{ code, msg, data }`，其 `code` 就是该文件中的业务码；插件 gRPC 接口不返回这些业务码。

## 插件管理 HTTP API

管理接口在受保护组内并要求管理员角色，路径为 `/api/v1/plugin/*`：安装、升级、重载、卸载、启用/停用、配置、运行时安装/切换/列表，以及页面代理与浏览器代理端点。**精确路由以** [`server/internal/router/routes/plugin/routes.go`](https://github.com/GOSpeak-org/GOSpeak-Server/blob/release/server/internal/router/routes/plugin/routes.go) **为准**，不在本页复述完整清单。

## 下一步

- [插件系统（使用指南）](/guide/plugin) — 安装、升级、运行时选版与安全警告
- [错误码](/reference/errors) — 平台 HTTP 业务错误码
- [API 参考](/reference/api) — 平台 HTTP API
