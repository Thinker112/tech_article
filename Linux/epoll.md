# epoll

## **epoll 介绍**

`epoll`（**event poll**）是 Linux 内核提供的高效 I/O 多路复用（I/O multiplexing）机制，专门用于处理大量并发连接的场景，如高并发服务器、网络代理等。它相对于传统的 `select` 和 `poll` 具有更高的性能和更好的扩展性。

------

## **1. epoll 的特点**

### **(1) 支持大量文件描述符**

- `select` 和 `poll` 受限于 `FD_SETSIZE`（通常为 1024），`epoll` 没有限制（仅受系统最大文件描述符数 `ulimit -n` 影响）。

### **(2) 事件驱动**

- `epoll` 采用 **事件触发**（event-driven）模式，不需要遍历整个监听列表，提高了效率。
- 触发模式分为：
  - **LT（水平触发，Level Triggered）**：只要文件描述符可读/可写，每次调用 `epoll_wait` 都会返回。
  - **ET（边缘触发，Edge Triggered）**：文件描述符从不可读变为可读、不可写变为可写时才触发一次，减少系统调用次数，提高性能。

### **(3) 内核优化**

- `epoll` 使用 **红黑树** 维护文件描述符，提高增删改查的效率（O(logN)）。
- 通过 **事件回调机制** 仅返回就绪的文件描述符，避免轮询遍历。

------

## **2. epoll 相关系统调用**

### **(1) epoll_create**

创建 `epoll` 实例，返回 `epoll` 句柄：

```c
int epoll_create(int size);
int epoll_create1(int flags);  // 推荐使用
```

示例：

```c
int epfd = epoll_create1(0);
if (epfd == -1) {
    perror("epoll_create1");
    exit(EXIT_FAILURE);
}
```

### **(2) epoll_ctl**

管理 `epoll` 监听的文件描述符，包括添加、修改、删除：

```c
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```

- ```
  op
  ```

   参数：

  - `EPOLL_CTL_ADD`：添加 `fd`
  - `EPOLL_CTL_MOD`：修改 `fd`
  - `EPOLL_CTL_DEL`：删除 `fd`

- `struct epoll_event` 结构：

```c
struct epoll_event {
    uint32_t events; // 监听的事件，如 EPOLLIN, EPOLLOUT
    epoll_data_t data; // 用户数据，可存储 fd 或其他信息
};
```

### **(3) epoll_wait**

等待事件发生：

```c
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

- `events`：存储触发事件的数组

- `maxevents`：一次最多返回多少事件

- ```
  timeout
  ```

  ：

  - `-1`：无限等待

  - `0`：立即返回

  - > 0：超时等待（毫秒）

示例：

```c
struct epoll_event events[10];
int nfds = epoll_wait(epfd, events, 10, -1);
if (nfds == -1) {
    perror("epoll_wait");
}
```

------

## **3. epoll 示例代码**

```c
#include <sys/epoll.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <errno.h>

#define MAX_EVENTS 10

int main() {
    int epfd = epoll_create1(0);
    if (epfd == -1) {
        perror("epoll_create1");
        exit(EXIT_FAILURE);
    }

    int fd = open("/tmp/testfile", O_RDONLY | O_NONBLOCK);
    if (fd == -1) {
        perror("open");
        exit(EXIT_FAILURE);
    }

    struct epoll_event ev;
    ev.events = EPOLLIN;
    ev.data.fd = fd;
    if (epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev) == -1) {
        perror("epoll_ctl");
        exit(EXIT_FAILURE);
    }

    struct epoll_event events[MAX_EVENTS];
    while (1) {
        int nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);
        for (int i = 0; i < nfds; i++) {
            if (events[i].events & EPOLLIN) {
                char buf[128];
                ssize_t n = read(events[i].data.fd, buf, sizeof(buf));
                if (n > 0) {
                    buf[n] = '\0';
                    printf("Read: %s\n", buf);
                }
            }
        }
    }

    close(fd);
    close(epfd);
    return 0;
}
```

------

## **4. epoll vs. select/poll**

| 特性       | select                            | poll               | epoll (Linux)             |
| ---------- | --------------------------------- | ------------------ | ------------------------- |
| 监听方式   | 数组                              | 链表               | 红黑树 + 回调             |
| 最大 fd 数 | 受 `FD_SETSIZE` 限制（默认 1024） | 无限制（但效率低） | 仅受系统 `ulimit -n` 限制 |
| 遍历方式   | 轮询整个集合                      | 轮询整个集合       | 仅返回就绪 fd             |
| 性能       | O(n)                              | O(n)               | O(1) 或 O(logN)           |
| 适合场景   | 低并发、小型应用                  | 中等并发           | 高并发服务器              |

------

## **5. 适用场景**

`epoll` 适用于：

- **高并发网络服务器**（如 Nginx、Redis、MySQL）
- **事件驱动模型**（如 WebSocket、消息队列）
- **大规模并发 I/O**（如 P2P 下载、爬虫）

------

## **6. 总结**

- `epoll` 是 Linux 提供的高效 I/O 事件通知机制，比 `select` 和 `poll` 更适合处理高并发。
- 采用 **事件驱动**（event-driven）机制，避免了无效遍历，提高性能。
- 适用于 **高并发网络编程**，如 Web 服务器、数据库、消息队列等。

# Linux I/O事件

在计算机操作系统中，**I/O 事件** 是指**与输入输出操作相关的状态变化**，通常由操作系统的 I/O 子系统检测并通知应用程序。例如，当一个文件描述符变得可读、可写，或者出现错误，内核会触发相应的 I/O 事件。

在 **epoll**、**select**、**poll** 等 I/O 多路复用机制中，I/O 事件通常用于监视网络 socket、文件、管道等的状态变化，以便程序可以高效地处理数据而不会陷入阻塞。

------

## **常见的 I/O 事件**

### **1. 读事件（Read Events）**

这些事件表示某个文件描述符（fd）有数据可读，通常用于监听网络 socket、管道、终端输入等。

- `EPOLLIN`

  （数据可读）：表示有数据可供读取，例如：

  - socket **收到** 新的数据（TCP 数据、UDP 数据）。
  - **标准输入**（`stdin`）有数据输入。
  - **管道**（pipe）、**文件描述符**（fd）有新数据可读。

- **`EPOLLRDHUP`**（对端关闭连接）：适用于 TCP 连接，表示对方关闭了写端（`shutdown(fd, SHUT_WR)`），但仍可读。

#### **示例**

监听 socket 是否有数据可读：

```c
struct epoll_event ev;
ev.events = EPOLLIN;  // 监听读事件
ev.data.fd = sockfd;
epoll_ctl(epfd, EPOLL_CTL_ADD, sockfd, &ev);
```

------

### **2. 写事件（Write Events）**

这些事件表示某个文件描述符（fd）可以写数据，通常用于：

- `EPOLLOUT`

  （可写）：表示 socket 

  可以写入数据

  ，不会阻塞。例如：

  - 当 socket 发送缓冲区有足够的空间时，`epoll_wait()` 返回 `EPOLLOUT`。
  - 适用于非阻塞 socket，确保写操作不会因为缓冲区满而阻塞程序。

#### **示例**

监听 socket 是否可写：

```c
struct epoll_event ev;
ev.events = EPOLLOUT;  // 监听写事件
ev.data.fd = sockfd;
epoll_ctl(epfd, EPOLL_CTL_ADD, sockfd, &ev);
```

------

### **3. 异常事件（Error Events）**

异常事件用于检测 socket 或文件描述符出现错误，防止程序在异常情况下进入死循环或崩溃。

- `EPOLLERR`

  （发生错误）：文件描述符发生错误，例如：

  - TCP 连接出现错误（如 `ECONNREFUSED`）。
  - 硬件设备错误（如磁盘 I/O 错误）。

- `EPOLLHUP`

  （挂起，Hangup）：对端关闭了连接，适用于：

  - TCP 连接被 **对方断开**（`close()`）。
  - 监听的管道（pipe）另一端关闭。

#### **示例**

检测 socket 是否异常：

```c
if (events[i].events & (EPOLLERR | EPOLLHUP)) {
    printf("Socket error or closed\n");
    close(events[i].data.fd);
}
```

------

## **I/O 事件示例：基于 `epoll` 的非阻塞 TCP 服务器**

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/epoll.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <string.h>

#define MAX_EVENTS 10

int main() {
    int epfd = epoll_create1(0);
    if (epfd == -1) {
        perror("epoll_create1");
        exit(EXIT_FAILURE);
    }

    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in addr = {
        .sin_family = AF_INET,
        .sin_port = htons(8080),
        .sin_addr.s_addr = INADDR_ANY
    };
    bind(listen_fd, (struct sockaddr*)&addr, sizeof(addr));
    listen(listen_fd, 10);

    struct epoll_event ev;
    ev.events = EPOLLIN;
    ev.data.fd = listen_fd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev);

    struct epoll_event events[MAX_EVENTS];
    while (1) {
        int nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);
        for (int i = 0; i < nfds; i++) {
            if (events[i].data.fd == listen_fd) {
                int client_fd = accept(listen_fd, NULL, NULL);
                ev.events = EPOLLIN | EPOLLET;  // 监听可读事件，使用边缘触发模式
                ev.data.fd = client_fd;
                epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &ev);
            } else if (events[i].events & EPOLLIN) {
                char buf[1024];
                int n = read(events[i].data.fd, buf, sizeof(buf));
                if (n <= 0) {
                    close(events[i].data.fd);
                } else {
                    printf("Received: %s\n", buf);
                }
            }
        }
    }

    close(listen_fd);
    close(epfd);
    return 0;
}
```

------

## **总结**

| I/O 事件     | 说明             | 适用场景             |
| ------------ | ---------------- | -------------------- |
| `EPOLLIN`    | 有数据可读       | 监听 socket 读取数据 |
| `EPOLLOUT`   | 可以写数据       | 发送数据，不会阻塞   |
| `EPOLLRDHUP` | 对端关闭连接     | 适用于 TCP 连接      |
| `EPOLLERR`   | 发生错误         | 网络异常、磁盘错误等 |
| `EPOLLHUP`   | 挂起（连接断开） | 对端关闭 socket      |

**关键点：**

1. `EPOLLIN` 适用于 **数据到来** 时的处理，如 TCP 服务器。
2. `EPOLLOUT` 适用于 **确保 socket 可写**，防止缓冲区满时阻塞写操作。
3. `EPOLLERR` 和 `EPOLLHUP` 用于处理连接异常，避免程序崩溃。
4. `EPOLLET`（边缘触发）适用于高性能场景，但需要非阻塞 `read/write`。

**`epoll` 结合 I/O 事件，使高并发服务器能够高效处理网络请求，避免阻塞，提高吞吐量。**