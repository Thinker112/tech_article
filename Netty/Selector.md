## **1. 什么是 Selector？**
`Selector` 是 **Java NIO（New I/O）** 提供的 **多路复用（Multiplexing）** 机制，允许 **单个线程监听多个 Channel**（如 `SocketChannel`）的 I/O 事件，从而提高系统的并发能力。

在传统的 **阻塞 I/O（BIO）** 模型中，每个连接都需要一个独立的线程处理，而 **NIO 的 `Selector`** 使得 **一个线程可以管理多个连接**，从而大幅提升性能。

## **2. Selector 如何监听多个 Channel 的事件？****`Selector` 监听多个 `Channel` 的核心步骤**
1. **创建 `Selector`**
   
   ```java
   Selector selector = Selector.open();
   ```
   
2. **注册多个 `Channel` 到 `Selector`**
    - **创建非阻塞的 `SocketChannel`**
    - **注册到 `Selector`，并指定监听的 I/O 事件**
   ```java
   ServerSocketChannel serverChannel = ServerSocketChannel.open();
   serverChannel.configureBlocking(false); // 必须设置为非阻塞
   serverChannel.bind(new InetSocketAddress(8080));
   serverChannel.register(selector, SelectionKey.OP_ACCEPT);
   ```

3. **使用 `select()` 方法监听 I/O 事件**
   ```java
   while (true) {
       selector.select(); // 阻塞，直到有事件发生
       Set<SelectionKey> selectedKeys = selector.selectedKeys();
       Iterator<SelectionKey> iterator = selectedKeys.iterator();
   
       while (iterator.hasNext()) {
           SelectionKey key = iterator.next();
           if (key.isAcceptable()) {
               handleAccept(key);
           } else if (key.isReadable()) {
               handleRead(key);
           }
           iterator.remove(); // 处理完后必须删除
       }
   }
   ```

---

## **3. `Selector` 监听的四种 I/O 事件**
当 `Channel` 注册到 `Selector` 时，可以指定监听的事件：
| 事件类型 | 常量 | 触发条件 |
|----------|------------------|------------------------------------|
| **连接就绪** | `SelectionKey.OP_CONNECT` | 客户端 `connect()` 连接完成 |
| **接收就绪** | `SelectionKey.OP_ACCEPT` | `ServerSocketChannel` 有新连接 |
| **可读** | `SelectionKey.OP_READ` | `SocketChannel` 有数据可读 |
| **可写** | `SelectionKey.OP_WRITE` | `SocketChannel` 可写入数据 |

注册多个事件：
```java
channel.register(selector, SelectionKey.OP_READ | SelectionKey.OP_WRITE);
```

---

## **4. 完整代码示例**
**一个简单的 NIO `Selector` 服务器**

```java
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.util.Iterator;
import java.util.Set;

public class NioServer {
    public static void main(String[] args) throws IOException {
        Selector selector = Selector.open(); // 1. 创建 Selector
        ServerSocketChannel serverChannel = ServerSocketChannel.open();
        serverChannel.bind(new InetSocketAddress(8080));
        serverChannel.configureBlocking(false);
        serverChannel.register(selector, SelectionKey.OP_ACCEPT); // 2. 监听连接

        System.out.println("服务器启动，监听端口 8080...");
        
        while (true) {
            selector.select(); // 3. 监听事件（阻塞等待）
            Set<SelectionKey> selectedKeys = selector.selectedKeys();
            Iterator<SelectionKey> iterator = selectedKeys.iterator();

            while (iterator.hasNext()) {
                SelectionKey key = iterator.next();
                iterator.remove(); // 处理完必须删除

                if (key.isAcceptable()) {
                    handleAccept(selector, key);
                } else if (key.isReadable()) {
                    handleRead(key);
                }
            }
        }
    }

    private static void handleAccept(Selector selector, SelectionKey key) throws IOException {
        ServerSocketChannel serverChannel = (ServerSocketChannel) key.channel();
        SocketChannel clientChannel = serverChannel.accept();
        clientChannel.configureBlocking(false);
        clientChannel.register(selector, SelectionKey.OP_READ); // 监听读事件
        System.out.println("新客户端连接：" + clientChannel.getRemoteAddress());
    }

    private static void handleRead(SelectionKey key) throws IOException {
        SocketChannel clientChannel = (SocketChannel) key.channel();
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        int bytesRead = clientChannel.read(buffer);

        if (bytesRead == -1) {
            clientChannel.close(); // 关闭连接
            return;
        }

        buffer.flip();
        String message = new String(buffer.array(), 0, bytesRead);
        System.out.println("收到消息：" + message);

        // 回写消息
        buffer.clear();
        buffer.put(("服务端收到：" + message).getBytes());
        buffer.flip();
        clientChannel.write(buffer);
    }
}
```

---

## **5. Selector 监听多个 Channel 的关键点**
**✅ 一个 `Selector` 可监听多个 `Channel`**

```java
channel1.register(selector, SelectionKey.OP_READ);
channel2.register(selector, SelectionKey.OP_WRITE);
channel3.register(selector, SelectionKey.OP_ACCEPT);
```

**✅ `select()` 方法用于监听事件**

```java
selector.select(); // 阻塞直到有 I/O 事件发生
```

**✅ `selectedKeys()` 取出所有就绪的 `Channel`**

```java
Set<SelectionKey> selectedKeys = selector.selectedKeys();
```

**✅ `iterator.remove()` 必须手动删除已处理的 `SelectionKey`**

```java
iterator.remove(); // 处理完后删除，避免重复处理
```

---

## **6. Selector vs 传统阻塞 I/O（BIO）**
| 对比项 | **传统 BIO（阻塞 I/O）** | **NIO + Selector（非阻塞 I/O）** |
|--------|----------------------|--------------------------------|
| 线程模型 | **每个连接一个线程** | **一个线程管理多个连接** |
| 资源消耗 | **线程数多，消耗大** | **线程数少，资源占用低** |
| 适用场景 | **低并发** | **高并发（如 Web 服务器）** |
| 事件处理 | **阻塞等待 I/O** | **基于 `Selector` 监听事件** |
| **示例** | **Servlet（BIO 版 Tomcat）** | **Netty / NIO 版 Tomcat / Kafka** |

---

## **7. 结论**
✅ **`Selector` 允许一个线程监听多个 `Channel`**，大幅提升高并发处理能力。  
✅ **适用于高并发服务器（如 Netty、Kafka、Tomcat NIO）**，比传统阻塞 I/O 更高效。  
✅ **配合 `SelectionKey` 可以监听 `ACCEPT`、`READ`、`WRITE` 事件，实现非阻塞通信。**

📌 **适用于：高性能网络服务器、聊天应用、RPC、微服务框架** 🚀