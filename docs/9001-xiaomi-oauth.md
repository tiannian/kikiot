## Xiaomi Home OAuth 授权流程说明 / Xiaomi OAuth Flow

本文从“协议与数据”的角度梳理 Xiaomi Home 在 Home Assistant 中执行 OAuth2 授权的流程，**刻意剥离具体 Python 实现细节**，只保留主要时序、字段拼接方式、URL 以及凭证存储方式，便于在其它语言/系统中复用。

---

### 1. 参与方与本地存储结构 / Parties & Local Storage

- **Home Assistant 实例**
  - 代表“一个运行中的 HA 环境”，有自己的全局实例 ID（记为 `ha_uuid`，可以是任意稳定 GUID）。

- **Xiaomi Home 集成实例**
  - 用户在 HA 中添加 Xiaomi Home 集成时，会创建一个配置实例：
    - 为该实例生成：
      - `virtual_did`：随机 64bit 整数或随机字符串，仅在本实例内部使用。
      - `uuid`：用于在 Xiaomi Cloud 中标识此 HA 设备。
    - 存储在 HA 的内部存储（通常是 `.storage/xiaomi_home`）中。

- **小米 OAuth2 服务**
  - 标准 OAuth2 授权码模式（Authorization Code Grant）。
  - Xiaomi Home 集成使用专用的 `client_id`（由小米分配给此集成，本文记为 `OAUTH2_CLIENT_ID`）。

- **配置常量来源与存储位置 / Where Key Params Are Stored**
  - `client_id`（即 `OAUTH2_CLIENT_ID`）：
    - 在集成源码 `miot/const.py` 中以常量形式写死（`OAUTH2_CLIENT_ID = '2882303761520251711'`），供 OAuth 客户端与 HTTP 客户端统一使用。
    - 不会作为用户可编辑配置保存在 HA 存储中。
  - `oauth_redirect_url`：
    - 配置向导中由用户选择/确认，默认值来自常量 `OAUTH_REDIRECT_URL`（定义在 `miot/const.py`，默认为 `http://homeassistant.local:8123`）。
    - 集成在运行时生成 `webhook_path` 后，将两者拼接得到 `redirect_uri`，并把完整值存入 HA 配置条目的 `data["oauth_redirect_url"]` 字段中（保存在 HA 的 `.storage/core.config_entries` 文件里）。
  - `DEFAULT_OAUTH2_API_HOST`：
    - 同样定义在 `miot/const.py` 中（`DEFAULT_OAUTH2_API_HOST = 'ha.api.io.mi.com'`）。
    - 在代码中根据 `cloud_server` 拼接得到最终的 OAuth2 / HTTP API Host，例如中国区直接使用 `ha.api.io.mi.com`，其它区域使用 `<cloud_server>.ha.api.io.mi.com`。

- **本地凭证存储结构（概念上）**
  - 以 `(uid, cloud_server)` 为键存储用户配置；每个键下至少包含：
    - `auth_info`：
      - `access_token`: 字符串，调用 HTTP API 和 MQTT 时使用。
      - `refresh_token`: 字符串，用于刷新 access token。
      - `expires_in`: 整数，token 有效期（秒）。
      - `expires_ts`: 整数时间戳，内部计算得到的“预期失效时间”（通常是当前时间 + `expires_in * 0.7`）。
      - （可能还有 `mac_key` 等其它签名相关字段）
  - 额外会缓存设备列表、家庭信息等，这里不展开。

---

### 2. 配置阶段：生成 OAuth 回调 URL / Config Phase: Building Redirect URL

1. **用户侧输入 / User Inputs**
   - 云区域：`cloud_server`，如 `cn`、`eu`、`us` 等。
   - 语言：`integration_language`，用于集成 UI 和错误文案。
   - OAuth 回调基础地址：`oauth_redirect_url`，例如：
     - `https://<home-assistant-base-url>`（集成内通常约束为固定预设值）。

2. **生成设备标识 `uuid`**
   - 集成使用如下规则生成一个设备级 UUID（伪代码）：
     - `uuid = sha256(ha_uuid + "." + virtual_did + "." + cloud_server)[0:32]`
   - 此 `uuid` 在进入 OAuth 流程时会被拼入 `device_id`：
     - `device_id = "ha." + uuid`

3. **生成最终 OAuth 回调 URL (`redirect_uri`)**
   - 为本次授权生成一个随机 webhook 路径（只需保证全局唯一），示例：
     - `webhook_path = "/api/webhook/<random-id>"`  
       （例如：`/api/webhook/5e2f1d21-8a9a-4f...`）
   - 将用户配置的基础地址与 webhook 路径拼接：
     - `redirect_uri = oauth_redirect_url + webhook_path`
   - 后续所有与小米 OAuth 服务器的交互，都会把这个 `redirect_uri` 作为回调地址上报。

---

### 3. 构造授权链接并发起 OAuth / Building the Authorization URL

1. **选择 OAuth 服务器域名**
   - 依据 `cloud_server` 选择 OAuth API Host：
     - 中国区：`oauth_host = DEFAULT_OAUTH2_API_HOST`
     - 其它区：`oauth_host = "<cloud_server>." + DEFAULT_OAUTH2_API_HOST`
   - OAuth 授权页面基础 URL（记为常量）：
     - `authorize_base_url = OAUTH2_AUTH_URL`

2. **生成 `state`**
   - 为防止 CSRF，集成基于 `device_id` 生成一个稳定的 `state` 值，逻辑如下（伪代码）：
     - `state = sha1("d=" + device_id)`
   - 该值会：
     - 放入授权 URL 的 query 参数。
     - 与当前授权会话一同保存，在回调校验时比对。

3. **授权 URL 的参数拼接 / Query Parameters**
   - 在调用浏览器打开授权页面前，拼接以下参数：
     - `redirect_uri`: 上文生成的 `redirect_uri`。
     - `client_id`: `OAUTH2_CLIENT_ID`。
     - `response_type`: `"code"`。
     - `device_id`: `"ha." + uuid`。
     - `state`: 上一步的 SHA1 字符串。
     - `scope`: 可选，当前实现中留空。
     - `skip_confirm`: `true`（授权有效期内且已登录时允许跳过授权确认页）。

   - URL 形态示例（伪代码）：
     - `https://<authorize_base_url>?redirect_uri=<url-encoded-redirect-uri>&client_id=<client-id>&response_type=code&device_id=ha.<uuid>&state=<sha1>&skip_confirm=true`

4. **本地会话状态**
   - 在发起授权前，集成会把至少如下信息保存在内存态会话中（以 `virtual_did` 为 key）：
     - 当前授权使用的 `state`。
     - 语言/文案对象（用于回调页本地化）。
     - 一个“等待授权结果”的占位（例如 Future/Promise）用于之后写入 `code`。

5. **用户交互**
   - 前端 UI 展示“前往小米授权”的链接或按钮，链接指向上述构造出来的授权 URL。
   - 用户在小米授权页完成登录与授权后，浏览器被重定向回 `redirect_uri`。

---

### 4. 回调处理：验证 `code` / `state` / Callback Handling

当用户完成授权后，小米会执行：

```text
GET <redirect_uri>?code=<auth-code>&state=<state>
```

对回调的处理逻辑可以抽象为：

1. **读取 query 参数**
   - 必须存在：
     - `code`：授权码。
     - `state`：防 CSRF 的随机值。

2. **校验 `state`**
   - 从本地会话（以 `virtual_did` 或其它会话 ID 识别）中取出原始的 `state`。
   - 若 `state_from_query != state_in_session`：
     - 视为非法请求，返回错误页，并终止流程。

3. **保存 `code`，唤醒等待方**
   - 将 `code` 写入本地会话（或直接唤醒等待者，如一个 Future/Promise）。
   - 授权流程的“后台任务”会在此处拿到 `code`，继续后续“交换 token” 的步骤。

4. **向浏览器返回页面**
   - 成功场景：返回一个简单 HTML 页面，提示“授权成功，可以关闭此页面并返回 HA”。
   - 失败场景：返回错误说明页面，提示用户重新点击授权链接重试。

---

### 5. 交换 Token：从 `code` 到 `access_token` / Token Exchange

回调把 `code` 写回本地会话后，后台会执行以下逻辑与小米服务器交互：

1. **Token 接口 URL**
   - 按区域选择 host：
     - `token_host = DEFAULT_OAUTH2_API_HOST`（中国区）
     - 或 `token_host = "<cloud_server>." + DEFAULT_OAUTH2_API_HOST`
   - 令牌接口固定路径：
     - `https://<token_host>/app/v2/ha/oauth/get_token`

2. **请求方式与参数封装**
   - HTTP 方法：`GET`
   - Query 参数示例：
     - `data = json.dumps({ "client_id", "redirect_uri", "code" 或 "refresh_token", "device_id" })`
     - 整体请求为：

```text
GET https://<token_host>/app/v2/ha/oauth/get_token?data=<url-encoded-json>
Content-Type: application/x-www-form-urlencoded
```

   - 说明：
     - 首次授权：`data` 中携带 `code` 字段。
     - 刷新授权：`data` 中携带 `refresh_token` 字段。

3. **响应结构（简化）**
   - HTTP 状态码：
     - `200`：继续解析。
     - `401`：表示未授权，视为 `refresh_token` 或 `code` 无效。
     - 其它非 200：视为调用失败。
   - JSON 结构（关键字段）：

```json
{
  "code": 0,
  "result": {
    "access_token": "<string>",
    "refresh_token": "<string>",
    "expires_in": 3600,
    ...
  }
}
```

   - 其中：
     - `code == 0` 才表示成功。
     - `expires_in` 为 access token 有效期（秒）。

4. **本地扩展字段：`expires_ts`**
   - 为了提前刷新 token，本地会额外计算：

```text
expires_ts = now + expires_in * 0.7
```

   - 即：把 token 视为在 70% 的剩余时间点就“过期”，到时触发刷新逻辑。

5. **持久化 `auth_info`**
   - 将上述 `access_token`、`refresh_token`、`expires_in`、`expires_ts` 以及可能的 `mac_key` 等字段打包为 `auth_info`。
   - 写入本地存储（按 `uid + cloud_server` 作为键）：
     - `storage[("uid", "cloud_server")]["auth_info"] = auth_info`

6. **拉取用户与设备信息（HTTP API 概要）**
   - 后续会基于 `access_token` 调用 Xiaomi Cloud HTTP API，例如：
     - 获取用户信息：
       - `GET https://open.account.xiaomi.com/user/profile?clientId=<client-id>&token=<access-token>`
     - 获取家庭 & 设备列表（路径为 `/app/v2/homeroom/gethome`、`/app/v2/home/device_list_page` 等，均以 `https://<token_host>` 为前缀）。
   - 通过这些接口解析出：
     - 用户 `uid`（账号唯一标识）。
     - 家庭/房间列表。
     - 设备列表（`did`、`urn`、`model` 等），并写入本地缓存。

7. **账号唯一性约束**
   - 集成层面保证：在同一区域（`cloud_server`）内，同一个 `uid` 只能创建一个集成实例。
   - 可以简单地把 `unique_id = cloud_server + uid` 写入 HA 的配置系统，用于去重。

8. **释放回调入口**
   - 首次授权成功后，可以注销本次授权用的 webhook（对应 `redirect_uri`）。

---

### 6. 运行期：Token 刷新策略 / Runtime Token Refresh Strategy

授权完成并把 `auth_info` 持久化之后，运行期需要自动刷新 token，避免频繁重新授权：

1. **初始化加载凭证**
   - 集成启动时，根据 `uid + cloud_server` 从本地存储读取 `auth_info`。
   - 使用其中的 `access_token` 初始化：
     - HTTP 客户端：后续所有到 Xiaomi Cloud 的 HTTPS 请求都加 `Authorization: Bearer<access_token>`。
     - MQTT 客户端：连接云端消息总线时使用该 token。

2. **定时刷新判断**
   - 任何时刻，当网络可用时，按照如下逻辑判断是否需要刷新：

```text
refresh_time = auth_info.expires_ts - now
if refresh_time <= 60 秒:
    触发 refresh_token 流程
else:
    在 refresh_time 秒后再检查
```

   - 注意：`expires_ts` 已经是预留了 30% 安全余量的时间点。

3. **刷新请求与更新存储**
   - 刷新时调用同一个 token 接口：
     - `https://<token_host>/app/v2/ha/oauth/get_token?data=...`
   - 只是 `data` 中使用 `refresh_token` 字段，而非 `code`。
   - 成功返回后：
     - 更新内存中的 `access_token`、`refresh_token`、`expires_in`、`expires_ts`。
     - 立即更新：
       - HTTP 请求头里的 `Authorization`。
       - MQTT 连接使用的 token（必要时重连）。
     - 把新的 `auth_info` 写回本地持久化存储。

4. **异常与回退**
   - 如果刷新失败（网络错误、401、响应结构异常等），应：
     - 在前端展示“请重新配置账号授权”的错误信息。
     - 保留旧的 `auth_info` 以便短期内重试或继续使用（视错误类型而定）。

---

### 7. 重新授权 / Re‑auth

在以下场景可能需要重新走一次 OAuth 授权流程：

- `refresh_token` 已过期或被吊销。
- 用户希望切换为另外一个小米账号。

重新授权时可以复用同样的设计：

1. 使用已有的 `uuid` 生成新的 `device_id`（仍为 `ha.<uuid>`）。
2. 为本次授权重新生成一个 webhook 回调 `redirect_uri`。
3. 按前述规则构造授权 URL -> 打开浏览器 -> 等待回调写入新的 `code`。
4. 调用 token 接口交换出新的 `access_token`/`refresh_token`。
5. 可选：调用用户信息接口验证新的凭证是否仍对应同一个 `uid`，若不同则视为“切换账号”。

---

### 8. 网络依赖检测 / Network Dependency Checks

为了在配置阶段就发现网络问题，集成提供了一个“网络依赖检测”选项，逻辑可抽象为：

- 若用户勾选“检查网络依赖”，则依次测试：
  - **OAuth2 授权页面可达性**：
    - `OAUTH2_AUTH_URL`（浏览器最终访问的授权基地址）。
  - **HTTP Token 接口可达性**：
    - `https://<cloud_server>.<DEFAULT_OAUTH2_API_HOST>/app/v2/ha/oauth/get_token`
  - **MIoT-Spec API 可达性**：
    - `https://miot-spec.org/miot-spec-v2/template/list/device`
  - **MQTT Broker 可用性**：
    - 尝试 TCP 连接：`<cloud_server>-<DEFAULT_CLOUD_BROKER_HOST>:8883`

- 任何一项失败都会在配置表单中返回相应错误码（诸如 `unreachable_oauth2_host`、`unreachable_http_host` 等），阻止继续配置，提示用户先修复网络环境。

---

### 9. 小结 / Summary

- **一次授权**：
  - 集成为每个 Xiaomi Home 实例生成 `virtual_did` 与 `uuid`，并基于此构造 `device_id = "ha." + uuid`。
  - 使用 `oauth_redirect_url + "/api/webhook/<random-id>"` 作为 `redirect_uri`，并在本地注册对应的回调处理。
  - 按约定拼接授权 URL：包含 `redirect_uri`、`client_id`、`response_type=code`、`device_id`、`state`、`skip_confirm` 等字段。
  - 回调中严格校验 `state`，然后用 `code` 调用 `https://<cloud-region-host>/app/v2/ha/oauth/get_token` 换取 `access_token` / `refresh_token`，并持久化到本地 `auth_info` 中。

- **长期运行**：
  - 运行期通过 `expires_ts` + `refresh_token` 机制定时刷新 `access_token`，始终保持 HTTP 和 MQTT 通道可用。
  - 若刷新失败或凭证失效，则通过 UI 提示用户重新发起 OAuth 授权。

如需在其它编程语言或环境中重用该 OAuth 流程，只需按本文描述实现：**设备标识生成、授权 URL 拼接、回调 `state` 校验、token 接口调用与本地存储结构（包含 `access_token` / `refresh_token` / `expires_ts`）** 即可，无需依赖任何具体的 Python 代码细节。

