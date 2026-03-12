## Xiaomi Home 设备枚举流程 / Device Enumeration Flow

本文从“协议与数据”的角度梳理 Xiaomi Home 在 Home Assistant 中枚举设备列表（Device List）的流程，**刻意剥离具体 Python 实现细节**，只保留主要时序、字段拼接方式、URL 以及设备信息在本地的组织方式，便于在其它语言/系统中复用。

---

### 1. 参与方与数据视图 / Parties & Data Views

- **Xiaomi Cloud HTTP API**
  - 用于拉取账户下的设备列表。
  - 关键接口路径：`/app/v2/home/device_list_page`（挂在对应区域的 Xiaomi Cloud Host 下）。

- **MIoT Central Hub Gateway（可选）**
  - 作为“本地网关”，通过 MQTT 协议提供局域网内设备列表。
  - 设备列表主题：`proxy/getDevList`（挂在网关的 MQTT 连接之上）。

- **MIoT LAN 客户端（可选）**
  - 直接通过局域网协议（基于本地 Central Hub 的 HTTP/自定义协议）获取设备列表。
  - 与 Cloud / Gateway 列表一起汇总到统一的“设备缓存”视图中。

- **MIoT Client（逻辑聚合层）**
  - 面向上层集成（如 Home Assistant）的统一入口。
  - 维护多个设备列表视角：
    - `device_list_cloud`：来自云端的设备信息。
    - `device_list_gateway`：来自本地 Central Hub 的设备信息。
    - `device_list_lan`：来自 LAN 扫描/控制的设备信息。
    - `device_list_cache`：最终对外暴露的统一设备列表视图。

---

### 2. 云端设备列表拉取流程 / Cloud Device List Enumeration

#### 2.1 HTTP Host 与 URL 拼接

- **Host 选择规则**
  - 记 `cloud_server` 为区域代码（`cn`、`eu`、`us` 等）。
  - 记 `DEFAULT_OAUTH2_API_HOST = "ha.api.io.mi.com"`。
  - Xiaomi Home 集成的 Cloud HTTP Host 与 OAuth Host 共享同一基础域，根据区域选择：
    - 中国区：`http_host = DEFAULT_OAUTH2_API_HOST`
    - 其它区域：`http_host = "<cloud_server>." + DEFAULT_OAUTH2_API_HOST`

- **设备列表接口 URL**
  - 完整 URL 形态：
    - `https://<http_host>/app/v2/home/device_list_page`

#### 2.2 请求参数结构 / Request Payload

- 请求方法：`POST`
- 请求体字段（JSON）：
  - `limit`: 整数，单页最大返回数量（示例：`200`）。
  - `get_split_device`: 布尔值，是否返回被分享/拆分的子设备。
  - `get_third_device`: 布尔值，是否包含第三方设备。
  - `dids`: `string[]`，可选；若为空列表表示“按账号查询全部设备”；若非空则表示仅查询指定 `did` 集合。
  - `start_did`: 字符串，可选；用于分页的起始设备 ID。

可以抽象为如下伪结构：

```json
{
  "limit": 200,
  "get_split_device": true,
  "get_third_device": true,
  "dids": ["<did-1>", "<did-2>", "..."],
  "start_did": "<optional-start-did>"
}
```

#### 2.3 返回结构与关键字段 / Response Structure

- HTTP 成功时，返回 JSON 对象，其中关键字段：

```json
{
  "result": {
    "list": [
      {
        "did": "device-id",
        "uid": "user-id",
        "name": "device-name",
        "spec_type": "urn:miot-spec-v2:device:xxx:yyyy",
        "model": "xiaomi.device.model",
        "pid": 0,
        "token": "optional-lan-token",
        "isOnline": true,
        "icon": "https://.../icon.png",
        "parent_id": "parent-device-id",
        "voice_ctrl": 0,
        "rssi": -45,
        "owner": { "...": "..." },
        "local_ip": "192.168.1.2",
        "ssid": "WiFi-SSID",
        "bssid": "AA:BB:CC:DD:EE:FF",
        "orderTime": 0,
        "extra": {
          "fw_version": "x.y.z"
        }
      }
    ],
    "has_more": false,
    "next_start_did": null
  }
}
```

- Xiaomi Home 集成会从单个设备条目中提取并映射为内部字段：
  - `did`: 设备唯一 ID（字符串）。
  - `uid`: 账号 ID。
  - `name`: 设备名称。
  - `urn`: 使用 `spec_type` 字段，MIoT-Spec URN，用于后续 spec 匹配。
  - `model`: 设备型号，如 `xiaomi.airpurifier.xxxx`。
  - `connect_type`: 由 `pid` 映射的连接类型（整数，语义由协议定义）。
  - `token`: LAN 控制所需 token（若存在）。
  - `online`: 从 `isOnline` 布尔值映射。
  - `icon`: 设备图标 URL。
  - `parent_id`: 若设备挂载在网关下，则为网关的 `did`。
  - `manufacturer`: 通常为 `model` 前缀（用第一个点号前的字符串）。
  - `voice_ctrl`: 语音控制能力标记（0 / 1 / 2 等）。
  - `rssi`: 信号强度。
  - `owner`: 所有者/分享者信息对象。
  - `pid`: 原始产品 ID。
  - `local_ip` / `ssid` / `bssid`: 局域网网络信息。
  - `order_time`: 排序时间戳。
  - `fw_version`: 从 `extra.fw_version` 中提取的固件版本号。

#### 2.4 分页时序 / Pagination Sequence

- 集成以递归或循环方式分页拉取所有设备：
  1. 以 `start_did = null` 调用接口，得到首批 `list` 与分页字段：
     - `has_more`: 布尔值，是否仍有后续设备。
     - `next_start_did`: 若 `has_more == true`，为下一页的起始 `did`。
  2. 将当前页中解析出的设备插入内部设备字典（以 `did` 为 key）。
  3. 若 `has_more == true` 且存在 `next_start_did`：
     - 使用同一 `dids` 参数与新的 `start_did` 重复步骤 1-2。
  4. 最终返回聚合后的 `did -> 设备信息` 映射。

---

### 3. 本地网关设备列表（Central Hub / MQTT）/ Gateway Device List

#### 3.1 设备列表 Topic 与请求

- 当本地 Central Hub 可用时，集成会通过其 MQTT 接口获取设备列表：
  - 主题：`proxy/getDevList`
  - 载荷：简单 JSON，如 `"{}"` 或包含筛选参数的 JSON 字符串（协议内部约定）。

- 返回结果中包含一个 `devList` 字段：

```json
{
  "devList": {
    "<did-1>": {
      "name": "device-name",
      "urn": "urn:miot-spec-v2:device:xxx:yyyy",
      "model": "xiaomi.device.model",
      "online": true,
      "specV2Access": true,
      "pushAvailable": true
    },
    "...": { "...": "..." }
  }
}
```

#### 3.2 字段过滤与映射

- 集成在处理 `devList` 时采用以下规则：
  - 若 `name` / `urn` / `model` 任意字段缺失，则忽略该设备。
  - 若 `model` 位于“不支持型号列表”（`UNSUPPORTED_MODELS`）中，则忽略该设备。
  - 对有效设备，映射为统一结构：

```json
{
  "did": "<did>",
  "online": true,
  "specv2_access": true,
  "push_available": true
}
```

- 映射后的结果以 `did` 为 key 汇总到一个字典中，供后续与云端列表合并。

#### 3.3 设备列表变更回调 / Device List Change Callback

- Central Hub 客户端在初始化时会注册一个“设备列表变更回调”：
  - 回调签名抽象为：`on_dev_list_changed(self, did_list: list[str])`。
  - 当网关检测到设备上下线或属性变化导致列表发生更新时，调用该回调。
  - 上层（MIoT Client）据此触发重新拉取相关设备信息并更新本地缓存。

---

### 4. 局域网设备列表（LAN 扫描）/ LAN Device List

#### 4.1 启用与前提

- 当 `MIoT LAN` 功能被启用且初始化成功时，集成会：
  - 向 LAN 客户端订阅设备状态变化；
  - 调用 `get_dev_list_async` 获取初始 LAN 设备列表。

#### 4.2 LAN 列表拉取逻辑

- 抽象调用时序：
  1. 检查 LAN 客户端是否已初始化完成（若未完成，则直接返回空列表）。
  2. 在内部事件循环线程中触发“获取设备列表”的实际调用。
  3. 使用一个 Future/Promise 在主循环中等待结果：
     - 当内部线程拿到设备列表结果后，在主循环线程上写入 Future 的结果。
  4. 最终返回设备列表 JSON 对象。

- 返回结构依赖于局域网协议实现，但从逻辑上等价于：

```json
{
  "<did-1>": { "...": "..." },
  "<did-2>": { "...": "..." }
}
```

---

### 5. 统一设备缓存与状态合并 / Unified Device Cache & State Merge

#### 5.1 内部缓存结构

- MIoT Client 内部维护多个设备列表：
  - `_device_list_cloud: { did: { ...云端字段... } }`
  - `_device_list_gateway: { did: { ...网关字段... } }`
  - `_device_list_lan: { did: { ...LAN 字段... } }`
  - `_device_list_cache: { did: { ...统一视图... } }`

- 对外公开属性：
  - `device_list`：直接暴露 `_device_list_cache`。

#### 5.2 设备在线状态合并 / Online State Merge

- 设备的“是否在线”由多源信息合并判定，抽象逻辑为：

```text
online_cloud   = device_list_cloud[did]?.online (True/False/None)
online_gateway = device_list_gateway[did]?.online (True/False/None)
online_lan     = device_list_lan[did]?.online (True/False/None)

online_final = some_merge_function(online_cloud, online_gateway, online_lan)
```

- 合并策略示例（简化）：
  - 若任意来源报告在线，则视为在线；
  - 某些实现可能优先信任局域网或网关状态。

- 当任一来源设备状态变化时（如 Cloud/MQTT/LAN 上线或下线）：
  1. 更新对应来源的设备字典（如 `_device_list_cloud[did]["online"] = True`）。
  2. 调用统一的“状态重计算函数”，得到新的 `online_final`。
  3. 若新旧状态不同，更新 `_device_list_cache[did]["online"]` 并通知上层。

#### 5.3 本地设备列表持久化 / Local Persistence

- Xiaomi Home 集成会在本地持久化设备列表缓存（由 `miot_storage` 管理），供重启后快速恢复。
- 持久化的键通常基于 `(uid, cloud_server)`，与 OAuth 凭证存储保持一致。
- 存储内容包含：
  - 设备的基本信息（`did`/`urn`/`model`/`name` 等）。
  - 最近一次判定的在线状态。
  - 部分网络信息（如 `local_ip`、`ssid`）与拓扑信息（如 `parent_id`）。

---

### 6. 设备列表变更通知 / Device List Change Notification

#### 6.1 触发条件

- 下列事件会触发“设备列表变化”通知逻辑：
  - 云端设备列表重新拉取，发现新增 / 删除 / 下线设备。
  - 网关上报的设备列表发生变更。
  - LAN 扫描或本地状态检测引发设备上线/下线。

#### 6.2 通知内容与本地存储

- 通知内容抽象为：
  - `device_list_add`: 新增设备列表（`did` + 设备名称）。
  - `device_list_del`: 不可用设备列表。
  - `device_list_offline`: 下线设备列表。
  - `network_status`: 当前网络状态（如 `online` / `offline`）。

- 这些内容通常通过 UI 或持久化通知渠道展示给用户（例如 Home Assistant 的持久化通知），**协议本身并不强制格式**，只要能表达上述语义即可。

---

### 7. 在其它语言/系统中复用的要点 / Reuse Checklist

- **HTTP 侧**
  - 按区域生成 `http_host`：
    - `cn` 使用 `ha.api.io.mi.com`。
    - 其它区域使用 `<cloud_server>.ha.api.io.mi.com`。
  - 使用 `POST https://<http_host>/app/v2/home/device_list_page`：
    - 请求体中包含 `limit`、`get_split_device`、`get_third_device`、可选 `dids` 与 `start_did`。
    - 按 `has_more` / `next_start_did` 做分页，聚合完整设备字典。
  - 从返回的 `result.list` 中提取必要字段，映射到统一结构。

- **网关 & LAN 侧（可选）**
  - 若支持本地 Central Hub：
    - 通过 MQTT 主题 `proxy/getDevList` 获取 `devList`。
    - 过滤无效/不支持设备后，记录 `online` / `specV2Access` / `pushAvailable`。
  - 若支持 LAN：
    - 依据本地协议实现一个 `get_dev_list` 调用，并以 `did -> info` 字典形式返回。

- **统一缓存与状态管理**
  - 按 `did` 合并 Cloud / Gateway / LAN 三路数据，形成统一的 `device_list_cache`。
  - 在线状态基于三路状态合并，变化时通知上层。
  - 将设备列表与 OAuth 凭证一起按 `(uid, cloud_server)` 维度持久化。

只要按本文描述复现上述 URL、字段结构与合并逻辑，即可在任何语言或运行环境中重建 Xiaomi Home 设备枚举能力，而无需依赖原始 Python 实现细节。

