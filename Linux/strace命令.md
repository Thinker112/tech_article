# strace命令

`strace` 是 Linux 下的一个调试工具，用于跟踪系统调用（syscalls）及其接收的信号。它可以帮助开发人员分析程序的运行行为，排查问题，甚至进行性能优化。

## 一、`strace` 的基本用法

### 1. 直接跟踪进程

```bash
strace ./my_program
```

这将会跟踪 `my_program` 的所有系统调用，并在终端输出详细信息。

### 2. 跟踪已运行进程

```bash
strace -p <PID>
```

用 `-p` 选项附加到一个已运行的进程（`PID` 为进程号）。

### 3. 将输出保存到文件

```bash
strace -o trace.log ./my_program
```

使用 `-o` 选项将 `strace` 的输出重定向到 `trace.log` 文件，方便分析。

------

## 二、常用选项

### 1. 只跟踪某类系统调用

```bash
strace -e open,read,write ./my_program
```

使用 `-e` 选项指定只跟踪 `open`、`read`、`write` 这几种系统调用。

### 2. 统计系统调用的执行时间

```bash
strace -c ./my_program
```

`-c` 选项会统计各个系统调用的执行次数、耗时等信息，而不会输出每个调用的详细日志。

### 3. 显示时间戳

```bash
strace -t ./my_program
```

`-t` 选项会在每个系统调用前显示时间戳。

### 4. 显示进程和线程 ID

```bash
strace -f -ff -p <PID>
```

- `-f` 选项：跟踪子进程（fork 出来的进程）。
- `-ff` 选项：为每个子进程创建单独的日志文件（如 `trace.1234`）。

### 5. 解析文件路径

```bash
strace -yy ./my_program
```

`-yy` 选项会解析 `open`、`stat` 等系统调用的文件路径，而不仅仅显示文件描述符。

------

## 三、应用示例

### 1. 调试程序崩溃

如果某个程序运行时崩溃或异常退出，可以使用 `strace` 来查找可能的原因：

```bash
strace ./faulty_program
```

它可能会显示 `open` 失败、`segmentation fault` 之类的错误信息，帮助定位问题。

### 2. 查看进程打开的文件

```bash
strace -e openat -p <PID>
```

这可以用于检查某个进程访问了哪些文件。

### 3. 分析网络请求

```bash
strace -e trace=network -p <PID>
```

可以用来查看进程的网络请求行为，如 `connect`、`sendto` 等调用。

------

## 四、结合 `ltrace` 使用

`strace` 跟踪的是系统调用，而 `ltrace` 则是跟踪库函数调用。例如：

```bash
ltrace ./my_program
```

如果 `strace` 没有显示明显的问题，可以用 `ltrace` 进一步分析。

------

## 五、总结

`strace` 是一个强大的 Linux 调试工具，主要用于：

- 追踪程序的系统调用和信号。
- 调试崩溃或错误行为。
- 分析程序的性能瓶颈。
- 监控进程对文件、网络等资源的访问。

在日常开发和运维中，熟练使用 `strace` 可以大大提高问题排查的效率。

# log分析

```bash
strace: Process 7579 attached
epoll_wait(46, [{events=EPOLLIN, data={u32=1581507968, u64=94087135094144}}], 512, -1) = 1
accept4(7, 0x7ffd874591a0, [112], SOCK_NONBLOCK) = -1 EAGAIN (Resource temporarily unavailable)
epoll_wait(46, [{events=EPOLLIN, data={u32=1581507728, u64=94087135093904}}], 512, -1) = 1
accept4(6, {sa_family=AF_INET, sin_port=htons(40988), sin_addr=inet_addr("172.27.37.33")}, [112 => 16], SOCK_NONBLOCK) = 10
epoll_ctl(46, EPOLL_CTL_ADD, 10, {events=EPOLLIN|EPOLLRDHUP|EPOLLET, data={u32=1581508928, u64=94087135095104}}) = 0
epoll_ctl(46, EPOLL_CTL_DEL, 6, 0x7ffd8745910c) = 0
epoll_ctl(46, EPOLL_CTL_ADD, 6, {events=EPOLLIN|EPOLLEXCLUSIVE, data={u32=1581507728, u64=94087135093904}}) = 0
epoll_wait(46, [{events=EPOLLIN, data={u32=1581508928, u64=94087135095104}}], 512, 60000) = 1
recvfrom(10, "GET /favicon.ico HTTP/1.0\r\nHost:"..., 1024, 0, NULL, NULL) = 385
openat(AT_FDCWD, "/var/www/favicon.ico", O_RDONLY|O_NONBLOCK) = -1 ENOENT (No such file or directory)
gettid()                                = 7579
write(44, "2025/02/22 10:37:51 [error] 7579"..., 237) = 237
writev(10, [{iov_base="HTTP/1.1 404 Not Found\r\nServer: "..., iov_len=159}, {iov_base="<html>\r\n<head><title>404 Not Fou"..., iov_len=100}, {iov_base="<hr><center>nginx/1.24.0 (Ubuntu"..., iov_len=62}, {iov_base="<!-- a padding to disable MSIE a"..., iov_len=402}], 4) = 723
write(45, "172.27.37.33 - - [22/Feb/2025:10"..., 224) = 224
close(10)                               = 0
```

------

## **1. 进程 7579 监听网络事件**

```
epoll_wait(46, [{events=EPOLLIN, data={u32=1581507968, u64=94087135094144}}], 512, -1) = 1
```

- `epoll_wait(46, ..., -1)`

  ：

  - 进程 7579 **使用 `epoll_wait()` 监听文件描述符 46**（`epoll` 实例）。
  - `-1` 表示 **无限等待**，直到有事件发生。
  - 结果 `= 1`，表示 **有 1 个文件描述符变为可读**（有新的网络连接或数据可用）。

------

## **2. 处理新连接**

```
accept4(7, 0x7ffd874591a0, [112], SOCK_NONBLOCK) = -1 EAGAIN (Resource temporarily unavailable)
```

- `accept4(7, ... SOCK_NONBLOCK)`

  ：

  - 进程尝试 **接受新的 TCP 连接**（`7` 是监听 socket）。
  - 结果 `-1 EAGAIN`：表示 **没有可用连接**，可能是一个瞬时事件（非阻塞模式下需要重试）。

```
epoll_wait(46, [{events=EPOLLIN, data={u32=1581507728, u64=94087135093904}}], 512, -1) = 1
accept4(6, {sa_family=AF_INET, sin_port=htons(40988), sin_addr=inet_addr("172.27.37.33")}, [112 => 16], SOCK_NONBLOCK) = 10
```

- 这次 

  ```
  accept4()
  ```

  成功

  ：

  - 接受了来自 `172.27.37.33:40988` 的新连接。
  - 返回的文件描述符是 **`10`**（表示已建立的连接）。

------

## **3. 更新 epoll 事件**

```
epoll_ctl(46, EPOLL_CTL_ADD, 10, {events=EPOLLIN|EPOLLRDHUP|EPOLLET, data={u32=1581508928, u64=94087135095104}}) = 0
```

- `epoll_ctl(EPOLL_CTL_ADD, 10, {EPOLLIN|EPOLLRDHUP|EPOLLET})`

  ：

  - **将新连接 (`fd=10`) 添加到 epoll 监视列表**。
  - 监听：
    - `EPOLLIN`（数据可读）
    - `EPOLLRDHUP`（对端关闭连接）
    - `EPOLLET`（边缘触发）

```
epoll_ctl(46, EPOLL_CTL_DEL, 6, 0x7ffd8745910c) = 0
epoll_ctl(46, EPOLL_CTL_ADD, 6, {events=EPOLLIN|EPOLLEXCLUSIVE, data={u32=1581507728, u64=94087135093904}}) = 0
```

- **删除并重新添加 `6` 号文件描述符**，可能是监听 socket 的优化调整。

------

## **4. 处理 HTTP 请求**

```
epoll_wait(46, [{events=EPOLLIN, data={u32=1581508928, u64=94087135095104}}], 512, 60000) = 1
recvfrom(10, "GET /favicon.ico HTTP/1.0\r\nHost:"..., 1024, 0, NULL, NULL) = 385
```

- `epoll_wait()` **发现新的可读数据**（文件描述符 `10`）。

- ```
  recvfrom(10, ...)
  ```

  读取 385 字节的 HTTP 请求

  ，请求内容包含：

  ```
  GET /favicon.ico HTTP/1.0
  ```

  - 这是客户端请求 `favicon.ico`（网页图标）。
  - `fd=10` 是刚建立的连接。

------

## **5. 服务器尝试查找文件**

```
openat(AT_FDCWD, "/var/www/favicon.ico", O_RDONLY|O_NONBLOCK) = -1 ENOENT (No such file or directory)
```

- `openat()` 试图 **打开 `/var/www/favicon.ico`**（网站根目录下的图标）。
- 结果 `ENOENT`（No such file or directory）：文件 **不存在**。

------

## **6. 记录日志**

```
gettid() = 7579
write(44, "2025/02/22 10:37:51 [error] 7579"..., 237) = 237
```

- `gettid()` 获取当前线程 ID（7579）。

- ```
  write(44, ...)
  ```

  将错误日志写入文件描述符 44

  ：

  ```
  2025/02/22 10:37:51 [error] 7579#0: *1 open() "/var/www/favicon.ico" failed (2: No such file or directory)
  ```

  - **表明 `favicon.ico` 资源不存在，导致 404 错误**。

------

## **7. 返回 404 响应**

```
writev(10, [{iov_base="HTTP/1.1 404 Not Found\r\nServer: "..., iov_len=159}, {iov_base="<html>\r\n<head><title>404 Not Fou"..., iov_len=100}, {iov_base="<hr><center>nginx/1.24.0 (Ubuntu"..., iov_len=62}, {iov_base="<!-- a padding to disable MSIE a"..., iov_len=402}], 4) = 723
```

- ```
  writev()
  ```

  向客户端发送 404 响应

  ：

  ```
  HTTP/1.1 404 Not Found
  Server: nginx/1.24.0 (Ubuntu)
  ```

  - **HTTP 404**（找不到 `favicon.ico`）。

------

## **8. 记录访问日志**

```
write(45, "172.27.37.33 - - [22/Feb/2025:10"..., 224) = 224
```

- ```
  write(45, ...)
  ```

  记录访问日志

  ：

  ```
  172.27.37.33 - - [22/Feb/2025:10:37:51] "GET /favicon.ico HTTP/1.0" 404 723
  ```

  - 说明 `172.27.37.33` 访问了 `favicon.ico`，返回 `404`。

------

## **9. 关闭连接**

```
close(10) = 0
```

- **服务器关闭 TCP 连接**（`fd=10`）。
- 结束本次请求处理。

------

## **总结**

1. **监听 `epoll_wait()` 等待事件**（新连接或数据）。
2. **`accept4()` 处理新连接**（`fd=10`）。
3. **`epoll_ctl()` 注册监听 `fd=10`**。
4. **`recvfrom()` 读取 HTTP 请求**（请求 `favicon.ico`）。
5. **`openat()` 查找 `/var/www/favicon.ico`，但文件不存在**。
6. **记录错误日志**（文件不存在）。
7. **`writev()` 发送 HTTP 404 响应**。
8. **记录访问日志**（请求 `favicon.ico` 失败）。
9. **关闭 TCP 连接**。

**整体来看，这是 Nginx 处理 HTTP 请求的标准流程**，但由于 `favicon.ico` 不存在，返回了 `404`。