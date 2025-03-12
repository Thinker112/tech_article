# To

在 SIP（Session Initiation Protocol）中，`To` 头部的 `tag` 主要用于 **唯一标识对话（Dialog）**，防止不同会话之间的混淆。是否需要 `tag` 取决于 **请求的类型和会话状态**。

------

## **1. To头部 tag的作用**

- `To` 头部中的 `tag` 用于 **区分不同的对话（Dialog）**。
- 在 **初始请求（如 `INVITE`）** 中，`To` 头部 **通常没有 `tag`**，因为对话尚未建立。
- 在 **响应（如 `200 OK`）或后续请求（如 `BYE`）** 中，`To` 头部 **必须包含 `tag`**，以表明该请求属于已建立的对话。

------

## **2. To头部 tag何时需要？**

| **SIP 请求 / 响应**            | **是否需要 `To` `tag`？** | **原因**                                      |
| ------------------------------ | ------------------------- | --------------------------------------------- |
| `INVITE`（初始请求）           | ❌ **不需要**              | 初始 `INVITE` 未建立对话，`To` 头部无 `tag`   |
| `1xx`（如 `180 Ringing`）      | ✅ **需要**                | 代理服务器/被叫用户可能会提前分配 `tag`       |
| `200 OK`（对 `INVITE` 的响应） | ✅ **需要**                | 确认对话建立，`tag` 确保后续请求归属正确对话  |
| `ACK`（对 `200 OK` 的确认）    | ✅ **需要**                | 需匹配 `200 OK` 里的 `To` `tag`               |
| `BYE`（终止会话）              | ✅ **需要**                | 需要匹配已建立的对话，否则对端无法识别会话    |
| `CANCEL`（取消 `INVITE`）      | ❌ **不需要**              | 只影响事务（Transaction），而非对话（Dialog） |
| `OPTIONS`（查询对端能力）      | ❌ **不需要**              | `OPTIONS` 不是对话型请求，不属于特定对话      |
| `REGISTER`（注册请求）         | ❌ **不需要**              | `REGISTER` 只用于注册，不涉及对话             |
| `SUBSCRIBE` / `NOTIFY`         | ✅ **需要**                | 这些请求创建持久对话，需要 `To` `tag`         |

------

## **3. To 头部 tag 的实际应用**

### **（1）初始 `INVITE` 请求（没有 `tag`）**

当 UAC（呼叫发起方）发送 `INVITE` 时，`To` 头部 **不会** 包含 `tag`：

```plaintext
INVITE sip:bob@domain.com SIP/2.0
Call-ID: 12345@alice-PC
From: <sip:alice@domain.com>;tag=abc123
To: <sip:bob@domain.com>  ← **没有 `tag`**
```

**原因**：`tag` 由 **被叫方（UAS）** 生成，UAC 发送 `INVITE` 时不会知道它的值。

------

### **（2）1xx 临时响应（可能有 `tag`）**

当 UAS 发送 `180 Ringing`，它可以提前分配 `tag`：

```plaintext
SIP/2.0 180 Ringing
Call-ID: 12345@alice-PC
From: <sip:alice@domain.com>;tag=abc123
To: <sip:bob@domain.com>;tag=xyz789  ← **现在有tag了**
```

**原因**：

- 如果 **`tag` 在 `180 Ringing` 就出现**，说明 UAS 已提前决定 `tag`，以便后续消息使用。
- 但如果 `180 Ringing` 没有 `tag`，UAS 可能会等到 `200 OK` 时才分配。

------

### **（3）`200 OK` 确认对话（必须有 `tag`）**

`To` 头部必须包含 `tag`，以确保 UAC 知道对话已建立：

```plaintext
SIP/2.0 200 OK
Call-ID: 12345@alice-PC
From: <sip:alice@domain.com>;tag=abc123
To: <sip:bob@domain.com>;tag=xyz789  ← **必须有tag**
```

**原因**：`200 OK` 确认会话成功，UAC 需要 `To` `tag` 进行后续交互（如 `ACK` 和 `BYE`）。

------

### **（4）`ACK` 确认 `200 OK`（必须带 `tag`）**

UAC 发送 `ACK` 以确认 `200 OK`，它 **必须** 包含 `To` `tag`：

```plaintext
ACK sip:bob@domain.com SIP/2.0
Call-ID: 12345@alice-PC
From: <sip:alice@domain.com>;tag=abc123
To: <sip:bob@domain.com>;tag=xyz789  ← **必须有tag**
```

**原因**：`ACK` 必须匹配 `200 OK` 里的 `To` `tag`，否则对端可能拒绝。

------

### **（5）`BYE` 终止会话（必须带 `tag`）**

UAC 或 UAS 任何一方发送 `BYE`，都必须带上 `To` `tag`：

```plaintext
BYE sip:bob@domain.com SIP/2.0
Call-ID: 12345@alice-PC
From: <sip:alice@domain.com>;tag=abc123
To: <sip:bob@domain.com>;tag=xyz789  ← **必须有tag**
```

**原因**：`BYE` 需要匹配 `200 OK` 确认的对话，如果 `tag` 不匹配，接收方会返回 `481 Call/Transaction Does not exist`

------

### **（6）`CANCEL` 请求（不需要 `tag`）**

如果 UAC 发送 `CANCEL` 取消 `INVITE`，`To` 头部 **不会有 `tag`**：

```plaintext
CANCEL sip:bob@domain.com SIP/2.0
Call-ID: 12345@alice-PC
From: <sip:alice@domain.com>;tag=abc123
To: <sip:bob@domain.com>  ← **没有tag**
```

**原因**：

- `CANCEL` 只取消 **事务（Transaction）**，而不是会话（Dialog）。
- 因为 `INVITE` 可能还没收到 `200 OK`，所以 `To` 里没有 `tag`。

------

## **4. To 头部 tag缺失的常见错误**

| **错误场景**                   | **错误行为**            | **可能后果**                            |
| ------------------------------ | ----------------------- | --------------------------------------- |
| `BYE` 请求缺少 `To` `tag`      | `BYE` 请求格式错误      | **481 Call/Transaction Does not exist** |
| `ACK` 请求缺少 `To` `tag`      | `ACK` 无法匹配 `200 OK` | INVITE 会话未确认，影响通话             |
| `INVITE` 初始请求带 `To` `tag` | `INVITE` 请求格式错误   | 可能被 UAS 拒绝                         |
| `CANCEL` 请求带 `To` `tag`     | `CANCEL` 请求格式错误   | 可能被 UAS 拒绝                         |

------

## **5. 总结**

1. **`To` 头部的 `tag` 在初始 `INVITE` 请求中不需要**，但在 `200 OK` 及后续请求（`ACK`、`BYE`）中必须使用。
2. **如果 `BYE` 缺少 `To` `tag`，可能导致 `481 Call/Transaction Does not exist` 错误**。
3. **`CANCEL` 只用于取消事务，不涉及对话，所以 `To` 头部无 `tag`**。

正确理解 `To` `tag` 的使用，有助于避免 SIP 会话管理中的常见错误。

# Via

## **SIP Via 头部的详细说明**

------

`Via` 头部是 SIP 消息的**路由信息**，用于跟踪请求在 SIP 网络中的**传输路径**，确保**正确的响应**返回给请求发起方。`Via` 头部在 **请求** 和 **响应** 中都有重要作用。

------

## **1. Via头部的基本结构**

`Via` 头部的格式如下：

```plaintext
Via: SIP/2.0/传输协议 发送方IP地址:端口;参数
```

常见示例：

```plaintext
Via: SIP/2.0/UDP 192.168.1.10:5060;branch=z9hG4bK12345678
```

它由以下部分组成：

| **字段**                 | **说明**                   | **示例**                 |
| ------------------------ | -------------------------- | ------------------------ |
| `SIP/2.0`                | SIP 版本号                 | `SIP/2.0`                |
| `UDP`                    | 传输协议（UDP/TCP/TLS/WS） | `UDP`                    |
| `192.168.1.10`           | 发送方 IP 地址或域名       | `192.168.1.10`           |
| `5060`                   | 发送方 SIP 端口号          | `5060`                   |
| `branch=z9hG4bK12345678` | 分支标识，用于唯一标识事务 | `branch=z9hG4bK12345678` |

------

## **2. Via头部的作用**

1. **记录 SIP 请求经过的服务器（代理、B2BUA）**
   - 每经过一个 SIP 代理（Proxy），代理会**在请求的 `Via` 头部顶部添加自己的地址**。
   - `Via` 头部中的所有地址形成一个**返回路径**，确保响应能正确回到请求发起方。
2. **确保 SIP 响应能够正确返回**
   - 响应时，服务器**必须逐步移除 `Via` 头部**，按照 `Via` 头部中的路径依次返回给前一个节点。
   - 这样可以避免响应被错误地路由到其他服务器。
3. **提供事务跟踪和负载均衡**
   - `branch` 参数用于唯一标识事务，避免事务冲突。
   - `received` 和 `rport` 参数可以用于 NAT 处理，确保数据能够正确穿越 NAT。

------

## **3. Via头部的参数**

`Via` 头部可以携带多个参数，不同参数用于不同的功能：

### **（1）`branch` 参数（事务标识）**

- `branch` 参数用于唯一标识一个 SIP 事务（Transaction）。

- 在 SIP 2.0 中，所有 `branch` 值 **必须以 `z9hG4bK` 开头**（固定前缀）。

- 例如：

  ```plaintext
  Via: SIP/2.0/UDP 192.168.1.10:5060;branch=z9hG4bK6197336353
  ```

- 作用：

  - 在 SIP 代理（Proxy）中，`branch` 可以用于区分不同事务，避免请求重复处理。
  - 在 `ACK` 或 `CANCEL` 请求中，`branch` 值 **必须与原始 `INVITE` 相同**，确保事务匹配。

### **（2）`received` 参数（NAT 处理）**

- `received` 参数用于记录**实际接收该请求的 IP 地址**（当 `Via` 头部中的地址与实际 IP 不同时）。

- 例如：

  ```plaintext
  Via: SIP/2.0/UDP 10.10.1.10:5060;branch=z9hG4bK12345678;received=203.0.113.1
  ```

- 作用：

  - 代理服务器可以通过 `received` 参数记录 NAT 外网地址。
  - 确保 SIP 响应能正确返回到 NAT 后的客户端。

### **（3）`rport` 参数（NAT 穿越）**

- `rport` 参数用于请求方指示服务器**返回响应时使用请求的实际源端口**。

- 例如：

  ```plaintext
  Via: SIP/2.0/UDP 192.168.1.10:5060;branch=z9hG4bK1234abcd;rport
  ```

- 当服务器收到此请求时，它会在响应中填写 rport 的实际值：

  ```plaintext
  Via: SIP/2.0/UDP 192.168.1.10:5060;branch=z9hG4bK1234abcd;rport=5060
  ```

- 作用：NAT 设备通常会更改源端口，`rport` 让服务器能正确响应 NAT 后的客户端。

------

## **4. Via头部的行为**

### **（1）请求时，`Via` 头部的添加**

当 SIP 请求发送时，每经过一个代理服务器（Proxy），该代理都会在**顶部**插入自己的 `Via` 头部：

```plaintext
INVITE sip:bob@domain.com SIP/2.0
Via: SIP/2.0/UDP 10.10.1.1:5060;branch=z9hG4bKproxy1
Via: SIP/2.0/UDP 192.168.1.10:5060;branch=z9hG4bKclient1
```

- 第一条 `Via` 是代理服务器 `10.10.1.1` 添加的。
- 第二条 `Via` 是最初的 UAC（用户代理客户端 `192.168.1.10`）添加的。

### **（2）响应时，`Via` 头部的移除**

当 `200 OK` 响应返回时，每经过一个代理，**该代理会移除自己添加的 `Via`**：

```plaintext
SIP/2.0 200 OK
Via: SIP/2.0/UDP 192.168.1.10:5060;branch=z9hG4bKclient1  ← 代理移除了自己的 `Via`
```

最终 `200 OK` 只包含 UAC（用户代理客户端）的 `Via`，确保响应能正确到达原始请求方。

------

## **5. Via头部错误导致的问题**

| **错误场景**        | **错误行为**             | **可能后果**                        |
| ------------------- | ------------------------ | ----------------------------------- |
| `Via` 头部缺失      | 代理未插入自己的 `Via`   | 代理无法接收响应，导致 SIP 请求失败 |
| `branch` 参数不唯一 | 代理复用了 `branch`      | 可能导致事务冲突，请求处理异常      |
| `rport` 参数未返回  | 服务器未正确填写 `rport` | NAT 后客户端无法接收响应            |
| `received` 地址错误 | 代理未正确记录 NAT 地址  | 可能导致响应发送到错误的 IP         |

------

## **6. Via头部的示例**

### **（1）正常 SIP 事务（无代理）**

```plaintext
INVITE sip:bob@domain.com SIP/2.0
Via: SIP/2.0/UDP 192.168.1.10:5060;branch=z9hG4bKabc123;rport
```

响应：

```plaintext
SIP/2.0 200 OK
Via: SIP/2.0/UDP 192.168.1.10:5060;branch=z9hG4bKabc123;rport=5060
```

------

### **（2）经过代理的 SIP 请求**

请求：

```plaintext
INVITE sip:bob@domain.com SIP/2.0
Via: SIP/2.0/UDP 10.10.1.1:5060;branch=z9hG4bKproxy1
Via: SIP/2.0/UDP 192.168.1.10:5060;branch=z9hG4bKclient1
```

响应：

```plaintext
SIP/2.0 200 OK
Via: SIP/2.0/UDP 192.168.1.10:5060;branch=z9hG4bKclient1
```

代理服务器 `10.10.1.1` 移除了自己的 `Via`，确保响应能正确返回给 UAC。

------

## **7. 总结**

1. **`Via` 头部用于跟踪 SIP 消息的传输路径**，确保响应能够正确返回。
2. **每个 SIP 代理都会在请求的 `Via` 头部列表中插入自己的地址**。
3. **响应返回时，代理会依次移除自己的 `Via` 头部**，确保最终响应正确送达。
4. **`branch` 参数必须唯一**，用于事务识别。
5. **`received` 和 `rport` 参数用于 NAT 处理**，确保 SIP 消息正确穿越 NAT。

正确理解 `Via` 头部的工作机制，有助于分析 SIP 信令交互，排查通信故障。