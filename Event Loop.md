# Event Loop

**Event Loop**（事件循环）是一种用于**异步编程**的机制，广泛应用于 JavaScript（Node.js）、Python（asyncio）、Rust（Tokio）、Go（Goroutine 调度器）等编程语言。它的核心思想是**非阻塞 I/O + 事件驱动**，避免了传统多线程编程中的锁竞争问题，提高了高并发处理能力。

- 事件（Event）是指异步任务的回调，如定时器、I/O、Promise
- 循环（Loop ）轮询任务队列，决定何时执行异步任务的回调

------

## **1. 什么是 Event Loop？**

Event Loop 是**单线程**的**事件驱动**架构，它允许程序在等待 I/O 操作（如文件读写、网络请求）时，不会阻塞整个进程，而是继续执行其他任务。其核心流程如下：

1. **注册异步任务**（如 I/O 操作、定时器、回调函数）。
2. **事件循环**等待异步任务完成。
3. **任务完成后，将回调函数推入队列**，等待执行。
4. **主线程从任务队列中取出任务执行**，然后继续循环。

简单来说，Event Loop 让**单线程**可以**同时处理多个任务**，提高程序的吞吐量。

------

## **2. Event Loop 的执行流程**

Event Loop 主要包含**事件监听**和**任务调度**，整个流程如下：

1. **执行同步代码**（Script execution）：
   - 先执行所有**同步代码**（包括函数调用、变量赋值等）。
   - 如果有**异步任务**，它们会被添加到相应的队列中（如微任务队列、宏任务队列）。
2. **检查微任务队列（Microtask Queue）**：
   - **执行所有微任务**（如 `Promise.then`、`process.nextTick`）。
   - 直到微任务队列为空，才进入下一步。
3. **执行宏任务（Macrotask Queue）**：
   - **取出一个宏任务执行**（如 `setTimeout`、`setImmediate`、I/O 事件）。
   - 然后回到**步骤 2**，继续执行微任务队列。
4. **重复上述步骤，直到所有任务完成**。

------

## **3. Event Loop 在不同环境中的实现**

不同编程语言的 Event Loop 机制有所不同：

### **3.1 Node.js 的 Event Loop**

Node.js 的 Event Loop 分为 **6 个阶段**，每个阶段都有不同的任务：

```
┌──────────────────────────┐
│    timers（定时器）      │ // setTimeout, setInterval
├──────────────────────────┤
│    I/O callbacks         │ // I/O 事件（TCP、FS等）
├──────────────────────────┤
│    idle, prepare         │ // 内部使用，几乎不涉及
├──────────────────────────┤
│    poll（轮询）         │ // 处理 I/O 事件
├──────────────────────────┤
│    check（setImmediate）│ // setImmediate 任务
├──────────────────────────┤
│    close callbacks      │ // 关闭事件，如 socket.on('close')
└──────────────────────────┘
```

- **Timers**：执行 `setTimeout`、`setInterval` 的回调。

- **I/O callbacks**：执行 I/O 相关的回调，如文件读取、数据库查询等。

- **Idle, prepare**：Node.js 内部使用，开发者一般不会接触。

- Poll

  ：

  - 处理 I/O 事件（如 HTTP 请求、文件读取）。
  - 如果没有 I/O 事件，则等待新的任务。

- **Check**：执行 `setImmediate` 任务（比 `setTimeout(0)` 先执行）。

- **Close callbacks**：处理 `close` 事件，如 `socket.on('close')`。

#### **示例：Node.js Event Loop 执行顺序**

```javascript
setTimeout(() => console.log('setTimeout'), 0);
setImmediate(() => console.log('setImmediate'));
process.nextTick(() => console.log('nextTick'));
Promise.resolve().then(() => console.log('Promise'));

console.log('Script Start');
```

**执行顺序：**

```
Script Start
nextTick
Promise
setTimeout / setImmediate (顺序取决于 I/O 任务)
```

- **`process.nextTick` 和 `Promise.then`** 属于 **微任务（Microtask）**，会在当前事件循环结束前执行。
- **`setTimeout` 和 `setImmediate`** 属于 **宏任务（Macrotask）**，执行顺序取决于 Node.js 的 I/O 机制。

------

### **3.2 浏览器的 Event Loop**

浏览器的 Event Loop 与 Node.js 略有不同，它将任务分为：

- **同步任务**：立即执行的代码（如 `console.log`）。

- 微任务（Microtasks）

  ：

  - `Promise.then()`
  - `MutationObserver`
  - `queueMicrotask()`

- 宏任务（Macrotasks）

  ：

  - `setTimeout`
  - `setInterval`
  - `setImmediate`（仅 Node.js）
  - `requestAnimationFrame`
  - I/O 任务（如 `fetch`）

#### **示例：浏览器 Event Loop**

```javascript
console.log('Start');

setTimeout(() => console.log('setTimeout'), 0);
Promise.resolve().then(() => console.log('Promise'));
console.log('End');
```

**执行顺序：**

```
Start
End
Promise
setTimeout
```

- **同步代码** 先执行。
- **微任务（Promise）** 在当前 Event Loop 结束前执行。
- **宏任务（setTimeout）** 在下一次 Event Loop 执行。

------

## **4. Event Loop 的优势**

1. **避免线程阻塞**：
   - 传统多线程模型中，I/O 操作可能会导致线程阻塞，影响整体性能。
   - Event Loop 采用**非阻塞 I/O**，不会影响主线程的执行。
2. **高并发**：
   - 通过**事件驱动**，即使在**单线程环境**下，也能处理**多个并发任务**。
   - 适用于 Web 服务器、数据库查询等高并发任务。
3. **减少锁竞争**：
   - 传统多线程编程需要使用**锁（Lock）\**来防止数据竞争，Event Loop 采用\**单线程模型**，避免了锁竞争问题。
4. **适用于 I/O 密集型任务**：
   - Event Loop **不会因 I/O 操作阻塞**，非常适合**网络请求、数据库查询、文件读取**等 I/O 任务。

------

## **5. 适用场景**

- 高并发 Web 服务器

  ：

  - Node.js 的 Event Loop 使得 Express/Koa 等 Web 框架可以高效处理请求。

- 实时应用

  ：

  - 如 WebSocket 服务器、在线游戏、聊天应用。

- 前端 UI 渲染

  ：

  - 浏览器的 Event Loop 负责 UI 事件（如 `click` 事件）、`requestAnimationFrame` 等。

------

## **6. 总结**

| 特性         | Event Loop 机制                        |
| ------------ | -------------------------------------- |
| **执行方式** | 事件驱动（Event-Driven）               |
| **线程模式** | 单线程（但可以异步执行任务）           |
| **任务分类** | 同步任务、微任务、宏任务               |
| **优点**     | 非阻塞、高并发、无锁竞争               |
| **适用场景** | I/O 密集型任务（Web 服务器、实时应用） |

Event Loop **通过异步任务队列和事件驱动**，使得单线程环境也能高效处理并发任务。理解其执行顺序（同步任务 → 微任务 → 宏任务）对于编写高性能的异步代码至关重要。

# JavaScript Event Loop

下面通过代码详细演示 **Event Loop** 的运行机制，包括 **同步任务、微任务（Microtask）、宏任务（Macrotask）** 的执行顺序。

## 1. Event Loop 执行顺序示例

```javascript
console.log("1. 同步代码开始执行");

setTimeout(() => {
    console.log("5. setTimeout 宏任务执行");
}, 0);

Promise.resolve().then(() => {
    console.log("3. Promise 微任务执行");
});

console.log("2. 同步代码执行结束");
```

### 执行流程

1. **先执行同步代码**
   - `console.log("1. 同步代码开始执行");`
   - `console.log("2. 同步代码执行结束");`
2. **遇到 `setTimeout`**
   - `setTimeout` 属于 **宏任务（Macrotask）**，被放入 **宏任务队列**，等待执行。
3. **遇到 `Promise.then`**
   - `Promise.then` 属于 **微任务（Microtask）**，被放入 **微任务队列**，等待执行。
4. **同步代码执行完毕，进入微任务队列**
   - `console.log("3. Promise 微任务执行");` 执行。
5. **微任务执行完毕，进入宏任务队列**
   - `console.log("5. setTimeout 宏任务执行");` 执行。

### **输出结果**

```
1. 同步代码开始执行
2. 同步代码执行结束
3. Promise 微任务执行
5. setTimeout 宏任务执行
```

## **2. 宏任务（Macrotask） vs. 微任务（Microtask）**

```javascript
console.log("1. 开始执行");

setTimeout(() => {
    console.log("5. 宏任务：setTimeout");
}, 0);

Promise.resolve().then(() => {
    console.log("3. 微任务：Promise.then");
});

setImmediate(() => {
    console.log("6. 宏任务：setImmediate");
});

process.nextTick(() => {
    console.log("2. 微任务：process.nextTick");
});

console.log("4. 结束执行");
```

### **执行流程**

1. **执行同步任务**

   ```
   console.log("1. 开始执行");
   console.log("4. 结束执行");
   ```

2. **遇到 `setTimeout`**

   - `setTimeout` 属于 **宏任务（Macrotask）**，放入 **下一轮 Event Loop 的宏任务队列**。

3. **遇到 `Promise.then`**

   - `Promise.then` 属于 **微任务（Microtask）**，放入 **当前微任务队列**。

4. **遇到 `setImmediate`**

   - `setImmediate` 属于 **宏任务（Macrotask）**，放入 **下一轮 Event Loop 的宏任务队列**。

5. **遇到 `process.nextTick`**

   - `process.nextTick` 属于 **微任务（Microtask）**，**优先于 `Promise.then` 执行**，所以会**最先执行**。

6. **执行所有微任务**

   - `console.log("2. 微任务：process.nextTick");`
   - `console.log("3. 微任务：Promise.then");`

7. **进入下一轮 Event Loop，执行宏任务**

   - `console.log("5. 宏任务：setTimeout");`
   - `console.log("6. 宏任务：setImmediate");`

### **输出结果**

```
1. 开始执行
4. 结束执行
2. 微任务：process.nextTick
3. 微任务：Promise.then
5. 宏任务：setTimeout
6. 宏任务：setImmediate
```

## **3. 事件循环中的 I/O 任务**

`setTimeout` 和 `setImmediate` 的执行顺序在 I/O 任务中会有所不同：

```javascript
const fs = require("fs");

fs.readFile(__filename, () => {
    setTimeout(() => console.log("3. setTimeout"), 0);
    setImmediate(() => console.log("2. setImmediate"));
});

console.log("1. 同步代码");
```

### **执行流程**

1. **同步代码执行**：

   ```
   console.log("1. 同步代码");
   ```

2. **`fs.readFile` 执行**

   - `fs.readFile` 是**异步 I/O 操作**，它会在**下一轮 Event Loop 的 Poll 阶段**完成后，执行回调函数。

3. **I/O 任务完成后，进入 Poll 阶段**

   - `setTimeout` 进入 **下一轮宏任务队列**（Timers 阶段）。
   - `setImmediate` **优先执行**，因为 `setImmediate` 属于 Check 阶段，Poll 阶段执行完就直接进入 Check 阶段。

4. **执行 `setImmediate`**

   ```
   console.log("2. setImmediate");
   ```

5. **执行 `setTimeout`**

   ```
   console.log("3. setTimeout");
   ```

### **输出结果**

```
1. 同步代码
2. setImmediate
3. setTimeout
```

## 4. await 在 Event Loop 中的行为

```javascript
async function asyncFunction() {
    console.log("1. asyncFunction 开始");
    await new Promise(resolve => setTimeout(resolve, 0));
    console.log("3. asyncFunction 结束");
}

console.log("2. 全局代码");
asyncFunction();
console.log("4. 全局代码执行结束");
```

### **执行流程**

1. **同步代码执行**

   ```
   console.log("2. 全局代码");
   ```

2. **调用 `asyncFunction`**

   ```
   console.log("1. asyncFunction 开始");
   ```

   - `await` 触发 **异步操作**，相当于**暂停执行**，然后返回**微任务（Promise）**。

3. **同步任务执行完毕**

   ```
   console.log("4. 全局代码执行结束");
   ```

4. **进入微任务队列，执行 `await` 之后的代码**

   ```
   console.log("3. asyncFunction 结束");
   ```

### **输出结果**

```
2. 全局代码
1. asyncFunction 开始
4. 全局代码执行结束
3. asyncFunction 结束
```

## **5. 总结**

| 任务类型                | 例子                                                         |
| ----------------------- | ------------------------------------------------------------ |
| **同步任务**            | 变量赋值、函数调用、`console.log`                            |
| **微任务**              | `Promise.then`、`process.nextTick`                           |
| **宏任务**              | `setTimeout`、`setImmediate`、I/O 任务                       |
| **Event Loop 执行顺序** | **同步任务 → 微任务（全部执行完）→ 宏任务（执行一个，回到微任务）** |

## **6. 结论**

1. **同步任务最先执行**，不会进入 Event Loop。
2. **微任务（Microtask）优先级高于宏任务（Macrotask）**，每轮 Event Loop 执行完**同步代码后，先执行所有微任务**，再执行一个宏任务。
3. **I/O 任务的 `setImmediate` 先执行，`setTimeout` 之后执行**。

## 宏任务与微任务

在 JavaScript 的事件循环（Event Loop）中，任务可以分为 **宏任务（MacroTask）** 和 **微任务（MicroTask）**。

### **1. 宏任务（MacroTask）**

宏任务通常指 **宿主环境（浏览器或 Node.js）调度的任务**，主要包括：

- `setTimeout`
- `setInterval`
- `setImmediate`（仅限 Node.js）
- `I/O 任务`（如文件读取、网络请求）
- `UI 渲染任务`（浏览器环境）
- `MessageChannel`
- `requestAnimationFrame`（浏览器环境）

宏任务在每次 **事件循环（Event Loop）迭代时被处理**，即主线程空闲时会取出一个宏任务执行，执行完后再进入微任务阶段。

### **2. 微任务（MicroTask）**

微任务通常指 **在当前宏任务执行完后立即执行的任务**，主要包括：

- `process.nextTick`（仅限 Node.js）
- `Promise.then`、`catch`、`finally`
- `queueMicrotask`
- `MutationObserver`（浏览器环境）

微任务在**当前宏任务执行完毕后立即执行，不需要等待下一次事件循环**。

------

### **执行顺序**

1. 先执行 **同步代码**（包括 `console.log`）。
2. 然后执行 **所有微任务**（`process.nextTick` 任务优先于 `Promise.then`）。
3. 再执行 **一个宏任务**，然后重复上述步骤。

------

### **总结**

- **同步任务 > 微任务 > 宏任务**
- `process.nextTick` **优先于** `Promise.then`
- `setTimeout` 的回调执行时机早于 `setImmediate`（除非在 I/O 操作后调用）

# Java Event Loop

在 Java 生态中，**Event Loop** 主要用于 **异步编程、非阻塞 I/O、多任务调度** 等场景

---

## **1. Netty（高性能网络框架）**
**Netty** 是一个高性能、异步事件驱动的 **NIO 网络通信框架**，广泛用于构建高并发的网络应用（如 RPC、IM、游戏服务器）。它内部采用 **Reactor 模型**，使用 **Event Loop** 进行 I/O 事件处理。

### **Netty Event Loop 机制**
- **EventLoopGroup** 负责管理多个 **EventLoop** 线程，每个线程负责处理多个 **Channel** 的 I/O 事件。
- **EventLoop** 维护一个 **事件队列（TaskQueue）**，不断从队列取出任务执行。
- **ChannelPipeline** 处理数据的入站（Inbound）和出站（Outbound）操作。

### **Netty 代码示例**
```java
EventLoopGroup bossGroup = new NioEventLoopGroup(1); // 负责接收客户端连接
EventLoopGroup workerGroup = new NioEventLoopGroup(); // 负责 I/O 事件处理

try {
    ServerBootstrap bootstrap = new ServerBootstrap();
    bootstrap.group(bossGroup, workerGroup)
        .channel(NioServerSocketChannel.class) // 使用 NIO
        .childHandler(new ChannelInitializer<SocketChannel>() {
            @Override
            protected void initChannel(SocketChannel ch) {
                ch.pipeline().addLast(new MyHandler()); // 处理 I/O 事件
            }
        });

    ChannelFuture future = bootstrap.bind(8080).sync();
    future.channel().closeFuture().sync();
} finally {
    bossGroup.shutdownGracefully();
    workerGroup.shutdownGracefully();
}
```
### **特点**
✅ **基于 Event Loop 机制**，高效管理 I/O 事件  
✅ **非阻塞 NIO（基于 Java NIO 的 Selector 机制）**  
✅ **适用于高并发网络通信（HTTP、WebSocket、RPC）**  

---

## **2. Java NIO（非阻塞 I/O）**
**Java NIO（New I/O）** 是 Java 原生提供的 **非阻塞 I/O 框架**，用于高性能网络编程。它使用 **Selector** 监听多个 Channel 的事件（如可读、可写），并采用 **Event Loop** 处理事件。

### **NIO Event Loop 机制**
- **Selector** 监听多个 **Channel**（如 `SocketChannel`）。
- **Event Loop 轮询 Selector**，检查哪些 Channel 准备好进行 I/O 操作。
- **任务队列** 处理其他异步任务（如 `ByteBuffer` 读写）。

### **NIO 代码示例**
```java
Selector selector = Selector.open();
ServerSocketChannel serverChannel = ServerSocketChannel.open();
serverChannel.configureBlocking(false);
serverChannel.register(selector, SelectionKey.OP_ACCEPT);

while (true) {
    selector.select(); // 事件轮询
    Set<SelectionKey> selectedKeys = selector.selectedKeys();
    Iterator<SelectionKey> iterator = selectedKeys.iterator();
    
    while (iterator.hasNext()) {
        SelectionKey key = iterator.next();
        if (key.isAcceptable()) {
            SocketChannel clientChannel = serverChannel.accept();
            clientChannel.configureBlocking(false);
            clientChannel.register(selector, SelectionKey.OP_READ);
        } else if (key.isReadable()) {
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            SocketChannel clientChannel = (SocketChannel) key.channel();
            clientChannel.read(buffer);
            System.out.println("Received: " + new String(buffer.array()).trim());
        }
        iterator.remove();
    }
}
```
### **特点**
✅ **基于 Selector 进行事件轮询（Event Loop）**  
✅ **支持大规模并发连接**，适用于 **高性能服务器**（如 Netty）  
✅ **核心组件**：`Selector`、`Channel`、`ByteBuffer`  

## **总结**

| **框架 / 技术** | **Event Loop 作用**      | **适用场景**  |
| --------------- | ------------------------ | ------------- |
| **Netty**       | 高性能 I/O 事件驱动      | 网络通信、RPC |
| **Java NIO**    | 基于 Selector 的事件轮询 | 高并发服务器  |