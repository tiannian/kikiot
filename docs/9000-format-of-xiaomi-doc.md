## Xiaomi Home 90xx 文档格式 / Format for Xiaomi 90xx Docs

所有以 `90xx` 开头的 Xiaomi Home 技术文档，只需要共同遵循下面几件事。

---

### 1. 刻意剥离实现细节 / Strip Implementation Details

- **只写协议与数据**：描述 URL、Host、Topic、请求/响应字段、本地存储结构、时序与流程。
- **刻意不写实现细节**：尤其不写具体 Python 代码、不绑定具体类名/函数名，最多使用伪代码说明算法或字段拼接。

---

### 2. 目标读者 / Target Audience

- **目标读者**：希望在**其它语言或运行环境**中重建相同能力的开发者。
- 文档要让读者在不知道现有 Python 实现的情况下，也能按文档描述完成协议对接与本地存储设计。

---

### 3. 文件命名 / File Naming

- 文件名形如：`90xx-short-topic-name.md`。
- `xx` 为两位数字，按主题递增或按模块分段。
- `short-topic-name` 使用短横线连接的**英文短语**，概括文档主题，例如：
  - `9001-xiaomi-oauth.md`
  - `9002-enum-device.md`
  - `9003-central-gateway-local.md`

