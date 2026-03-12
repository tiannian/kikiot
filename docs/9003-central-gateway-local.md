## Xiaomi Home 本地中央网关控制 / Central Gateway Local Control

本文从“协议与数据”的角度梳理 Xiaomi Home 在 Home Assistant 中基于**本地中央网关（Central Hub Gateway）**实现设备本地控制的逻辑，**刻意剥离具体 Python 实现细节**，只保留主要时序、字段拼接方式、URL / 主题结构以及设备信息在本地的组织方式，便于在其它语言 / 系统中复用。

---

## 概览

- **控制路径优先级**
  - **优先使用本地中央网关控制**（Central Hub Gateway，上行 / 下行都走局域网 MQTT Broker）。
  - 如中央网关不可用，再尝试 **局域网直连控制（LAN 控制）**。
  - 如本地两种路径都不可用，最终退回 **云端控制（Cloud 控制）**。
- **核心角色**
  - **MIoT Network**：探测网络连通性、维护本机网卡信息。
  - **MIoT mDNS Service（MipsService）**：通过 mDNS 在局域网发现「小米中央网关服务」，每个家庭对应一个 `group_id`。
  - **MipsLocalClient（中央网关本地客户端）**：面向某个 `group_id` 的 MQTT / TLS Client，用于：
    - 获取该家庭下的 **设备列表**。
    - 订阅设备属性变更 / 事件。
    - 向设备下发本地控制指令。
  - **MIoTLan（LAN 控制）**：在没有中央网关时，直接通过 UDP 广播 + AES 加密报文与 IP 设备通信。
  - **MIoT Cloud HTTP / MQTT**：作为兜底控制与状态源。

---

## 中央网关发现与连接

### 1. mDNS 服务发现

1. 集成初始化时，会创建一个 **mDNS 浏览器**，监听局域网中的“小米中央网关服务”。
2. 每个中央网关对应一个 **服务分组 ID：`group_id`**，与米家家庭一一对应：
   - 配置阶段，用户在「选择家庭与设备」界面选中的每个家庭，都会保存对应的 `group_id` 与 `home_name`。
   - 这些家庭 / 房间 / 设备与 `group_id` 的映射信息来自 **MIoT Cloud HTTP API**，由 `miot_cloud.get_homeinfos_async()` 通过云端 RPC 调用  
     `POST /app/v2/homeroom/gethome` 获取（参数示例：`limit=150, fetch_share=true, fetch_share_dev=true, plat_form=0, app_ver=9`），
     并在本地整理为包含 `home_id` / `home_name` / `room_info` / `group_id` / `dids` 等字段的结构，供「选择家庭与设备」界面使用。
3. mDNS 浏览器持续维护以下信息：
   - 每个 `group_id` 对应的一条网关服务记录：
     - 网关 IP 地址列表：`addresses[]`
     - 网关服务端口：`port`
   - 当某个 `group_id` 的服务新增 / 更新 / 删除时，触发「服务状态变更回调」。

### 2. 基于 `group_id` 建立本地网关连接

1. 若当前控制模式允许本地控制，并且当前区域支持中央网关：
   - 对每一个已选择的家庭 `home_id`，取出其 `group_id` 与 `home_name`。
2. 对于已经通过 mDNS 扫描到服务记录的 `group_id`：
   - 构造一个 **本地 MQTT 客户端实例（`MipsLocalClient`）**，其关键配置为：
     - `did`：集成用于标识自身的 **虚拟设备 ID**（一个 64bit 整数，以字符串形式表示）。
     - `group_id`：家庭分组 ID。
     - `host`：中央网关在局域网内的 IP（取 mDNS 中地址列表的第一个）。
     - `port`：中央网关 MQTT/TLS 服务端口（来自 mDNS）。
     - `ca_file` / `cert_file` / `key_file`：用于 TLS 双向认证的本地证书路径。
       - `ca_file`：米家云内置的 CA 根证书，集成在本地首次使用时将内嵌 PEM 写入  
         `.storage/xiaomi_home/cert/mihome_ca.cert`，**不依赖任何 OAuth 接口获取**。
       - `key_file`：用于中央网关双向认证的用户私钥，由集成本地使用 Ed25519 随机生成，  
         并写入 `.storage/xiaomi_home/cert/{uid}_{cloud_server}.key`，**同样不通过云端接口下发**。
       - `cert_file`：用户证书，由米家云端在 OAuth 授权后，根据本地生成的 CSR 进行签发：  
         - 集成使用 OAuth2 获取到的 `access_token` 调用  
           `POST https://<cloud_server>.account.xiaomi.com/app/v2/ha/oauth/get_central_crt`，  
           请求体中携带 base64 编码的 CSR 字符串；  
         - 云端返回的 `result.cert` 为 PEM 格式证书，保存到  
           `.storage/xiaomi_home/cert/{uid}_{cloud_server}.cert`，即 `cert_file`；  
         - 证书过期前由集成定期调用同一接口刷新。
     - `home_name`：仅用于日志和 UI 展示。
3. 为每个 `MipsLocalClient` 注册回调：
   - **`on_dev_list_changed(group_id, did_list)`**：
     - 当中央网关侧认为某些设备列表发生变化时，发送设备 DID 列表。
   - **`sub_mips_state(group_id, state)`**：
     - 监控该网关连接上下线状态（`state = true/false`）。
4. 调用 `connect()` 建立与中央网关的 **MQTT/TLS 长连接**。

### 3. 网关服务变更时的重连策略

当同一 `group_id` 的 mDNS 记录发生变更（例如 IP / 端口改变）时：

1. 若当前已有对应的 `MipsLocalClient` 且其 client 标识 / IP / 端口与新记录一致：
   - 直接忽略，不做重连。
2. 否则：
   - 断开并销毁旧的 `MipsLocalClient`；
   - 按新的 IP / 端口 / 证书重新创建一个新的 `MipsLocalClient` 并连接。

当某个 `group_id` 所有服务记录被移除时：

- 视为该家庭的中央网关在局域网不可达，后续设备在线状态与消息订阅都会进行降级处理（详见后文设备状态合并）。

---

## 网关设备列表获取与本地组织

### 1. 设备列表获取入口

中央网关设备列表有两种触发方式：

- **全量刷新**
  - 在某个 `group_id` 的本地 MQTT 连接建立成功后，由集成主动触发「刷新该 `group_id` 下所有设备列表」。
- **增量刷新**
  - 当网关发送「设备列表变更通知」时，带上一个 `did_list`（设备 ID 列表），表示这些设备需要重新拉取详细信息。

### 2. 获取设备列表的请求负载

对某个 `group_id` 所在的中央网关发起设备列表查询时，构造的请求负载为一个 JSON 对象：

```json
{
  "filter": {
    "did": ["did1", "did2", "..."]   // 可选，若为空则表示全量查询
  },
  "info": [
    "name",
    "model",
    "urn",
    "online",
    "specV2Access",
    "pushAvailable"
  ]
}
```

- `filter.did`：
  - **为空或缺省**：请求该网关下的全部设备。
  - **为列表**：仅请求指定设备的详情。
- `info` 指定需要返回的字段：
  - `name`：设备名称。
  - `model`：设备型号。
  - `urn`：MIoT-Spec-V2 设备实例 URN。
  - `online`：设备是否在线（由网关维护）。
  - `specV2Access`：网关是否支持对该设备使用 MIoT-Spec-V2 接口进行本地访问。
  - `pushAvailable`：是否可通过网关推送属性 / 事件消息至 Home Assistant。

中央网关返回的 `gw_list` 结构为：

```json
{
  "<did>": {
    "name": "xxx",
    "model": "yyy",
    "urn": "urn:miot-spec-v2:device:...",
    "online": true,
    "specV2Access": true,
    "pushAvailable": true
  },
  "...": { ... }
}
```

### 3. 本地网关设备表结构

集成在内存中维护三份设备表（按来源区分）：

- **缓存总表：`device_list_cache : { did -> info }`**
  - 这是对外统一使用的「设备总表」，既包含云端信息，也会合并中央网关 / 局域网直连的信息。
  - 关键字段（示例）：
    - `did`：设备 ID（字符串）。
    - `name`：设备名称。
    - `model`：设备型号。
    - `urn`：MIoT-Spec-V2 设备实例 URN。
    - `group_id`：所属中央网关家庭 ID（如果存在）。
    - `online`：**合并计算**后的设备在线状态（见后文“设备状态合并规则”）。
    - `token` / `connect_type`：用于 LAN 控制 / 发现时下发到 `MIoTLan`。

- **云端设备表：`device_list_cloud : { did -> info }`**
  - 主要记录云端视角的 `online` 状态及云侧返回的基础信息。

- **中央网关设备表：`device_list_gateway : { did -> info }`**
  - 主要字段：
    - 与网关返回的 `gw_list[did]` 的内容一致。
    - 增加 `group_id` 字段，以标识该设备是由哪个家庭的中央网关提供本地控制。

在更新网关设备表时，逻辑大致为：

1. 对于在 `device_list_cache` 中已存在、且同时出现在 `gw_list` 中的 DID：
   - 更新该 DID 的：
     - `group_id`（如有变更）。
     - 网关视角的 `online` / `pushAvailable` / `specV2Access` 等字段。
2. 对于在 `device_list_cache` 中存在但不在 `gw_list` 中的 DID：
   - 将对应 `device_list_gateway[did].online` 标记为 `false`。
3. 对于只出现在 `gw_list` 中、尚未在缓存表中的 DID：
   - 先更新 `device_list_gateway`，再根据需要补充到缓存表（例如，设备刚刚被用户从米家 App 加入家庭）。

所有对 `device_list_cache` 的更新操作完成后，会将该表序列化，以 `domain='miot_devices'` / `name='<uid>_<cloud_server>'` 的形式持久化到本地存储。

---

## 设备状态合并与消息订阅来源选择

### 1. 云 / 网关 / LAN 三源合并规则

对任意设备 DID，集成会维护三个来源的「在线状态」：

- `cloud_state`：来自云端 MQTT 的在线状态（部分设备类型默认视为在线，例如 BLE、proxy 子设备）。
- `gw_state`：来自中央网关的在线状态（`device_list_gateway[did].online`）。
- `lan_state`：来自 LAN 控制（`device_list_lan[did].online`）。

合并规则为：

- 若 `cloud_state is None` 且 `gw_state == False` 且 `lan_state == False`：
  - 视为 **设备已被移除**，统一的 `device_list_cache[did].online = None`。
- 若任一来源为 `True`：
  - 统一的 `online = True`。
- 否则：
  - 统一的 `online = False`。

### 2. 消息订阅来源选择优先级

对于「属性变更 / 事件」的实时订阅，集成会为每个 DID 维护一个 **当前消息订阅来源 `sub_from`**，其优先级如下：

1. **中央网关（`sub_from = group_id`）**
   - 条件：
     - 网关侧设备在线：`device_list_gateway[did].online == true`。
     - 支持本地 MIoT-Spec-V2 接口：`device_list_gateway[did].specV2Access == true`。
     - 网关端可以推送消息：`device_list_gateway[did].pushAvailable == true`。
2. **LAN 控制（`sub_from = 'lan'`）**
   - 条件：
     - `device_list_lan[did].online == true`。
     - `device_list_lan[did].push_available == true`。
3. **云端（`sub_from = 'cloud'`）**
   - 条件：
     - `device_list_cloud[did].online == true`。

当任一来源状态变化时（云 / 网关 / LAN 上线 / 下线），都会触发对该 DID 的 `sub_from` 重新计算：

- 如果新的 `sub_from` 与旧的相同（且是 `'cloud'` 或 `'lan'` 这种全局点），则不做任何调整。
- 否则：
  - 先对旧来源执行「取消订阅」（属性、事件）。
  - 再对新来源执行「订阅」：
    - 云端：在云侧 MQTT 上订阅该 DID 的属性 / 事件主题。
    - 网关：在对应的 `MipsLocalClient(group_id)` 上订阅该 DID 的属性 / 事件。
    - LAN：通过 `MIoTLan` 的属性 / 事件订阅接口订阅该 DID 的广播消息。

---

## 中央网关与 LAN 控制的协同关系

### 1. 没有中央网关时的行为

当 mDNS 服务列表为空，且全局有至少一个可用网卡接口时：

- 集成会尝试初始化 `MIoTLan`：
  - 监听指定网卡上的 UDP 端口（默认 54321）。
  - 周期性向广播地址（例如 `255.255.255.255`）发送探测报文，用于发现支持 MIoT LAN 协议的设备。
- 此时本地控制路径为：
  - **LAN 控制 > 云端控制**（仍会结合云端状态）。

### 2. 有中央网关时的行为

当 mDNS 发现至少一个中央网关服务时：

- `MIoTLan` 会被显式 **停用（deinit）**：
  - 释放 LAN 控制相关的 socket / 线程 / 定时器。
  - 清空 LAN 设备表。
- README 中的规则也明确说明：
  - **在同一局域网内存在中枢网关时，即便用户开启了「局域网控制」功能，LAN 控制路径也不会生效。**
- 此时本地控制路径为：
  - **中央网关控制 > 云端控制**。

### 3. mDNS 网关服务消失 / 恢复时的切换

当 mDNS 监听到「中央网关服务消失」：

- 若当前没有任何网关服务存在：
  - 触发对 `MIoTLan` 的初始化流程，使 LAN 控制重新可用。

当之后重新发现中央网关服务：

- 再次停用 `MIoTLan`，并重新依托中央网关进行本地控制。

---

## 中央网关上的设备控制与消息格式（抽象）

> 本小节仅从抽象层面描述消息形态，不涉及具体 MQTT 主题名或 JSON 字段细节。具体主题与编码可参考官方协议文档。

### 1. 属性读写（Property）

- **读属性**
  - 通过网关向目标设备发起「属性读取」请求。
  - 请求中至少包含：
    - 设备 DID：`did`
    - 服务 ID：`siid`
    - 属性 ID：`piid`
  - 返回结果中包含：
    - 对应的 `value` 字段。

- **写属性**
  - 通过网关向目标设备发起「属性写入」请求。
  - 请求中包含：
    - `did` / `siid` / `piid` / `value`。
  - 返回结果中包含：
    - `code`：执行结果码（`0` 或 `1` 视为成功）。
  - 若网关返回的结果码表示失败，则集成会根据错误码映射为本地错误信息。

### 2. 方法执行（Action）

- 通过中央网关调用某设备的 Action 时，负载中包含：
  - `did` / `siid` / `aiid` / `in`（输入参数列表）。
- 返回结果中包含：
  - `code`：执行结果码。
  - `out`：输出参数列表（可选）。

### 3. 属性变更 / 事件上报

- 当中央网关订阅到设备的属性变更或事件上报时，会在本地向集成转发：
  - 属性变更：
    - 包含 `did` / `siid` / `piid` / `value`。
  - 事件上报：
    - 包含 `did` / `siid` / `eiid` / `arguments`（事件参数列表）。
- 集成会根据 `<did>/p/<siid>/<piid>` 或 `<did>/e/<siid>/<eiid>` 的**逻辑主题键**在本地匹配订阅者（实体），并投递给对应的处理逻辑。

---

## 小结：在其它系统复用的要点

- **发现阶段**
  - 使用 mDNS 在局域网中发现「小米中央网关服务」，以 `group_id` 标识家庭。
  - 为每个 `group_id` 建立一个面向网关的 MQTT/TLS 客户端。
- **设备列表阶段**
  - 通过一个 JSON 负载 `{ filter.did, info[] }` 获取网关设备列表。
  - 在本地维护三份设备表：云端、中央网关、LAN，并合并到统一的缓存总表。
- **状态与订阅阶段**
  - 根据云 / 网关 / LAN 的在线状态综合得出设备最终 `online` 状态。
  - 按优先级为每个 DID 选择消息订阅来源（中央网关 > LAN > 云），并在来源变化时自动切换订阅。
- **控制阶段**
  - 属性读写与 Action 调用统一抽象为：携带 `did` / `siid` / `piid` / `aiid` 以及 `value` / `in` / `out` 的 RPC 调用。
  - 错误码由网关 / 云端返回，再在本地做统一映射。
- **与 LAN 控制的协同**
  - 局域网内一旦存在中央网关，即关闭 LAN 控制，所有本地控制通过中央网关完成。
  - 无中央网关时才启用 LAN 控制，并作为云端控制的本地加速路径。

按照以上抽象，只要在目标系统中提供：

- mDNS 发现、
- MQTT/TLS 客户端、
- HTTP 客户端与本地存储能力，

就可以不依赖具体 Python 代码，完整复现「基于本地中央网关的米家设备控制」这一逻辑。 

