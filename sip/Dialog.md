# Dialog

#### **SIP 对话（Dialog）概述**

`Dialog`表示两个用户代理（User Agent，简称 UA）之间的一种点对点 SIP 关系，并在一定时间内持续存在。对话通常用于用户代理管理状态，但对于代理服务器来说通常不相关。它主要用于在用户代理之间进行消息排序和正确的请求路由。此外，`Dialog`为解释 SIP 事务（Transaction）和消息（Message）提供了上下文，但处理消息并不一定需要对话。

#### **对话的标识**

在每个用户代理（UA）中，对话由**对话 ID（Dialog Id）** 唯一标识，该 ID 由以下三部分组成：

1. **Call-ID**（通话 ID）
2. **本地标签（Local Tag）**
3. **远端标签（Remote Tag）**

对话 ID 在两个用户代理中是不相同的，具体来说，一个用户代理的本地标签（Local Tag）与对端用户代理的远端标签（Remote Tag）相同。标签（Tag）是用于生成唯一对话 ID 的不透明令牌（Opaque Token）。

#### **对话中包含的重要数据**

对话包含以下数据，用于支持后续消息传输：

- **对话 ID（Dialog Id）** - 用于唯一标识对话。
- **本地序列号（Local Sequence Number）** - 用于对 UA 发送给对端的请求进行排序。
- **远端序列号（Remote Sequence Number）** - 用于对对端发送给 UA 的请求进行排序。
- **本地 URI（Local URI）** - 本地用户的地址。
- **远端 URI（Remote URI）** - 远端用户的地址。
- **远端目标（Remote Target）** - 取自 `Contact` 头字段，可能来自请求、响应或刷新请求/响应。
- **安全标志（Secure Boolean）** - 指示是否使用 `sips:` 方案进行安全通信。
- **路由集（Route Set）** - 一个有序的 URI 列表，表示向对端发送请求时需要经过的服务器列表。

#### **对话的状态机**

对话具有状态机，其当前状态由初始对话中的消息序列决定：

- INVITE 请求的对话状态转换

  ```
  Null --> Early --> Confirmed --> Terminated
  ```

- 其他创建对话的请求（如 SUBSCRIBE）

  ```
  Null --> Confirmed --> Terminated
  ```

#### **ACK 处理**

- 监听器（Listener）**必须** 对 `2xx` 响应发送 `ACK`。
- 如果监听器未能立即发送 `ACK`，系统将自动终止该对话，并将其从 SIP 堆栈中移除。
- 在收到后续 `INVITE` 请求的 `2xx` 响应时，对话层会自动处理 `ACK` 的重传。

#### **错误响应**

- 当 [RFC 3261](https://www.ietf.org/rfc/rfc3261.txt) 规范要求 UA 必须返回某个特定错误时，对话层必须自动返回错误响应（如 500 服务器内部错误）。
- 这种情况下，`SipListener` 不会收到 `RequestEvent`，因为对话层会直接构造并发送错误响应。

#### **示例：RFC 3261 第 14 章**

如果一个用户代理服务器（UAS）在发送 `INVITE` 请求的最终响应之前，收到另一个 `CSeq` 序列号更高的 `INVITE`，它必须返回 `500 Server Internal Error`，并包含 `Retry-After` 头字段，值为 0 到 10 秒之间的随机数。

#### **响应的重传机制**

- 在此规范版本中，如果事务（Transaction）与某个对话（Dialog）相关联，则由对话层处理 `Response` 的重传。
- 如果没有关联的对话，则应用程序可选择接收重传提醒，并手动进行 `Response` 的重传（见 `ServerTransaction#enableRetransmissionAlerts()`）。
- 当启用重传提醒时，`SipProvider` 将向 `SipListener` 发送 `Timeout.RETRANSMIT` 通知，监听器可据此决定是否重传响应（`SipListener#processTimeout(TimeoutEvent)`）。

#### **INVITE 客户端事务（Client Transaction）**

- **UAC 需要 ACK 第一个 `2xx` 响应**。
- 如果 `2xx` 响应关联了对话，ACK 由对话层自动处理。
- 如果 `2xx` 响应未关联对话，监听器需手动 ACK，否则对话会在超时后终止。

#### **INVITE 服务器事务（Server Transaction）**

- **`2xx` 响应**
  - 应用程序通过 `ServerTransaction#sendResponse(Response)` 发送 `2xx` 响应。
  - 如果 `ServerTransaction` **没有关联对话**，且应用程序启用了超时提醒（`enableRetransmissionAlerts()`），`SipListener` 将周期性收到超时提醒，以手动重传 `2xx` 响应。
  - 如果 `ServerTransaction` **关联了对话**，则 SIP 堆栈会自动重传 `2xx` 响应，直到收到 `ACK`，`SipListener` 不会收到重传提醒。
- **`300-699` 响应**
  - 响应可以由应用程序或对话层发送。
  - `300-699` 响应的重传由事务层（Transaction Layer）自动处理，与是否存在对话无关。

#### **可靠的临时响应（Reliable Provisional Responses）**

- 可靠的临时响应始终由**对话层**发送，并由其负责重传。
- 应用程序无需处理这些响应的重传：
  - 应用程序通过 `Dialog#sendReliableProvisionalResponse(Response)` 发送可靠的临时响应。
  - SIP 堆栈会按照指数回退（Exponential Backoff）策略自动重传，直到收到 `PRACK` 或事务超时。

#### **处理 Forking 的 INVITE**

- 由于 SIP 代理可能会对 `INVITE` 进行分叉（Forking），UAC 可能会收到多个 `2xx` 响应，每个响应的 `TO` 头部字段的 `tag` 值不同。
- 每个 `2xx` 响应对应一个**新的**对话（Dialog），即它们具有不同的对话 ID。
- 在此情况下：
  - **第一个 `2xx` 响应终止原始 INVITE 事务**。
  - 额外的 `2xx` 响应将作为 `ResponseEvent` 提供给 `SipListener`，且 `ClientTransaction ID` 为空，但 `Dialog` 仍然有效。
  - 监听器必须对额外的 `2xx` 进行 `ACK`，否则对话将在超时后终止。
  - 如果**未禁用自动创建对话**，则 `ResponseEvent` 仍会包含 `Dialog`，无论 `INVITE` 是通过 `ClientTransaction` 还是 `SipProvider` 无状态（Stateless）发送的。

------

### **总结**

1. SIP 对话用于管理用户代理之间的消息状态和路由信息。
2. 对话通过 `Call-ID`、`Local Tag` 和 `Remote Tag` 唯一标识。
3. `ACK` 处理对 `2xx` 响应至关重要，否则对话可能终止。
4. 事务（Transaction）与对话（Dialog）协同工作，以保证 SIP 消息的可靠传输。
5. `INVITE` 的分叉处理会导致多个 `2xx` 响应，每个响应都可能创建新的对话。
6. 可靠的临时响应和 `2xx` 响应的重传机制由 SIP 堆栈自动管理。

这样，SIP 堆栈能够高效地管理对话，确保通信的稳定性和可靠性。

在 SIP（Session Initiation Protocol）中，**Dialog（对话）**和**Transaction（事务）**是两个不同的概念，它们用于描述 SIP 消息的不同级别的交互关系。

------

# **1. SIP Transaction（事务）**

## **1.1 事务的定义**

**SIP 事务**（Transaction）是 SIP 消息交互的最小单位，它由**一个请求及其对应的所有响应**组成。

## **1.2 事务的组成**

一个 SIP 事务通常包括：

- **客户端事务（Client Transaction）**：从请求的发送方（UAC，User Agent Client）到服务器（UAS，User Agent Server）。
- **服务器事务（Server Transaction）**：从服务器（UAS）接收请求并返回响应。

### **示例：一个完整的事务**

```plaintext
INVITE sip:bob@domain.com SIP/2.0  ← UAC 发送请求
↓
SIP/2.0 100 Trying  ← UAS 响应
↓
SIP/2.0 180 Ringing  ← UAS 响应
↓
SIP/2.0 200 OK  ← UAS 响应
```

在这个过程中：

- **一个 INVITE 事务**由 `INVITE` 请求和 `100 Trying`、`180 Ringing`、`200 OK` 响应组成。
- **一个 ACK 事务**是 `ACK` 请求（无响应）。

### **SIP 事务的两大类型**

SIP 事务分为 **两大类**：

1. **非 INVITE 事务**（Non-INVITE Transaction）

   - 例如 `REGISTER`、`BYE`、`OPTIONS`、`CANCEL` 等请求及其对应的响应。
   - 这些事务是**单轮交互**，即请求后直接获得最终响应。

   **示例：BYE 事务**

   ```plaintext
   BYE sip:bob@domain.com SIP/2.0  ← 终止通话的请求
   SIP/2.0 200 OK  ← 确认 BYE
   ```

2. **INVITE 事务**（INVITE Transaction）

   - `INVITE` 事务通常需要多轮交互，例如 `100 Trying`、`180 Ringing`、`200 OK` 。
   - `ACK` 不属于 `INVITE` 事务，而是一个独立的事务。

------

# **2. SIP Dialog（对话）**

## **2.1 对话的定义**

**SIP 对话**（Dialog）是**由多个事务组成的较长会话**，用于跟踪两个用户代理（UA）之间的通信状态。

**对话的关键点**

- **一个对话通常包含多个事务**（例如 `INVITE` 事务、`BYE` 事务）。
- **对话由 `Call-ID`、`To` 标签（`To tag`）、`From` 标签（`From tag`） 唯一标识**。

## **2.2 对话的建立**

对话通常由 `INVITE` 请求创建，并在 `200 OK` 响应时**正式建立**。

### **示例：一个完整的对话**

```plaintext
UAC → UAS：INVITE (创建对话)  ← 事务 1
UAS → UAC：100 Trying
UAS → UAC：180 Ringing
UAS → UAC：200 OK  (对话正式建立)
UAC → UAS：ACK (确认 200 OK) ← 事务 2
---
通话进行中
---
UAC → UAS：BYE (终止对话) ← 事务 3
UAS → UAC：200 OK (确认终止)
```

- **INVITE 事务**：包含 `INVITE`、`100 Trying`、`180 Ringing`、`200 OK`，用于创建对话。
- **ACK 事务**：独立的 `ACK` 事务（不属于 `INVITE` 事务）。
- **BYE 事务**：用于终止对话。

## **2.3 对话的唯一标识**

SIP **对话的唯一标识** 由以下 3 个字段组成：

1. **Call-ID**：SIP 头部中的 `Call-ID`，用于唯一标识一场通话。
2. **From Tag**：`From` 头部中的 `tag`，标识 UAC 端的会话实例。
3. **To Tag**：`To` 头部中的 `tag`，标识 UAS 端的会话实例。

示例：

```plaintext
Call-ID: 98765@192.168.1.100
From: <sip:alice@domain.com>;tag=123456
To: <sip:bob@domain.com>;tag=abcdef
```

以上 `Call-ID`、`From Tag` 和 `To Tag` 组合在一起，唯一标识该 SIP 对话。

------

# **3. Transaction vs. Dialog（事务 vs. 对话）**

| **对比项**   | **Transaction（事务）**      | **Dialog（对话）**              |
| ------------ | ---------------------------- | ------------------------------- |
| **作用**     | 处理一个 SIP 请求及其响应    | 维持整个会话的状态              |
| **组成**     | 一个请求及其所有响应         | 多个事务                        |
| **唯一标识** | `Via` 头部中的 `branch` 参数 | `Call-ID`、`From tag`、`To tag` |
| **生命周期** | 一个请求到最终响应结束       | 直到 `BYE` 终止会话             |
| **示例**     | `INVITE` 事务、`BYE` 事务    | `INVITE` + `BYE` 形成的完整通话 |

------

# **4. 事务和对话的常见问题**

## **（1）为什么 ACK 不属于 INVITE事务？**

- `ACK` 不会触发新的响应，而是直接送达对方，因此它是一个**独立的事务**。
- `INVITE` 事务在 `200 OK` 发送后就结束了，而 `ACK` 之后对话仍然存在。

## **（2）CANCEL 请求会创建对话吗？**

- **不会**，`CANCEL` 只是终止一个**进行中的 `INVITE` 事务**，但不会影响对话。
- 如果 `INVITE` 事务已经建立 `200 OK`，则 `CANCEL` 无效，需要 `BYE` 终止对话。

## **（3）为什么 BYE 事务可以终止对话？**

- `BYE` 事务属于已经建立的对话，并会导致对话的终结。
- `BYE` 结束后，`Call-ID` 仍然存在，但对话本身已经失效。

------

# **5. 结论**

1. **Transaction 事务**
   - 事务是 SIP 消息的最小交互单元，一个事务包含**一个请求及其所有响应**。
   - `INVITE` 事务和 `BYE` 事务是独立的。
   - `ACK` 是一个独立的事务，不属于 `INVITE` 事务。
2. **Dialog 对话**
   - 对话是一个更长的会话状态，通常由 `INVITE` 事务创建，并由 `BYE` 事务终结。
   - 由 `Call-ID`、`From tag` 和 `To tag` 共同标识。

**简单总结**：

- **事务 = 请求 + 响应**
- **对话 = 多个事务组成的通话会话**
- **`INVITE` 事务创建对话，`BYE` 事务终结对话**