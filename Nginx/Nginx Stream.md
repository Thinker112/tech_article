# Nginx Stream

## Nginx Stream 模块介绍

### 1. 概述

Nginx 的 `stream` 模块用于处理 **TCP** 和 **UDP** 流量的代理转发（Layer 4 代理），类似于传统的 **L4 负载均衡**。相比 HTTP 代理（Layer 7），`stream` 模块能够代理非 HTTP 协议的流量，如 MySQL、Redis、SSH、DNS、SMTP 以及基于 WebSocket 的应用等。

### 2. 启用 Stream 模块

`stream` 模块并非默认启用，需在编译 Nginx 时加上 `--with-stream` 选项：

```sh
./configure --with-stream
make && make install
```

部分 Linux 发行版的 Nginx 预编译包已包含 `stream` 模块，可通过以下命令检查：

```sh
nginx -V 2>&1 | grep -- '--with-stream'
```

如果包含 `--with-stream`，说明该 Nginx 版本已支持 `stream` 模块。

### 3. 配置示例

#### **3.1 TCP 代理**

下面的示例将所有到 `192.168.1.100:3306` 的 MySQL 连接转发到 `10.0.0.10:3306`：

```nginx
stream {
    upstream mysql_backend {
        server 10.0.0.10:3306;
        server 10.0.0.11:3306 backup;
    }

    server {
        listen 3306;
        proxy_pass mysql_backend;
    }
}
```

- `upstream mysql_backend`：定义后端 MySQL 服务器池，支持主备（`backup`）模式。
- `proxy_pass`：将 TCP 流量代理到 `mysql_backend`。

#### **3.2 UDP 代理**

以下示例用于代理 DNS（UDP 53 端口）请求：

```nginx
stream {
    upstream dns_backend {
        server 8.8.8.8:53;
        server 8.8.4.4:53;
    }

    server {
        listen 53 udp;
        proxy_pass dns_backend;
    }
}
```

- `listen 53 udp`：指定监听 UDP 53 端口。
- `proxy_pass dns_backend`：转发 DNS 查询请求到 `8.8.8.8` 和 `8.8.4.4`。

### 4. 关键指令解析

| 指令                      | 说明                                                  |
| ------------------------- | ----------------------------------------------------- |
| `stream {}`               | 定义 TCP/UDP 代理的全局配置。                         |
| `upstream name {}`        | 定义后端服务器组，可用于负载均衡。                    |
| `server {}`               | 定义流量代理规则（监听端口、协议等）。                |
| `listen [port] [udp/tcp]` | 指定监听的端口及协议（默认 TCP）。                    |
| `proxy_pass [backend]`    | 指定上游服务器（可使用 `upstream` 组或直接指定 IP）。 |
| `proxy_connect_timeout`   | 代理连接超时时间。                                    |
| `proxy_timeout`           | 代理会话超时时间（无数据时）。                        |
| `proxy_buffer_size`       | 代理缓冲区大小，适用于 TCP 应用优化。                 |

### 5. 负载均衡策略

`stream` 模块支持以下负载均衡方式：

- 轮询（默认）

  ：

  ```nginx
  upstream backend {
      server 10.0.0.1:6379;
      server 10.0.0.2:6379;
  }
  ```

- 权重

  ：

  ```nginx
  upstream backend {
      server 10.0.0.1:6379 weight=3;
      server 10.0.0.2:6379 weight=1;
  }
  ```

- 最少连接

  ：

  ```nginx
  upstream backend {
      least_conn;
      server 10.0.0.1:6379;
      server 10.0.0.2:6379;
  }
  ```

- 哈希（基于客户端 IP）

  ：

  ```nginx
  upstream backend {
      hash $remote_addr consistent;
      server 10.0.0.1:6379;
      server 10.0.0.2:6379;
  }
  ```

### 6. 日志配置

默认 `access_log` 仅适用于 HTTP 模块，但 `stream` 也支持独立的日志记录：

```nginx
stream {
    log_format basic '$remote_addr [$time_local] '
                     '$protocol $status $bytes_sent $bytes_received '
                     '$session_time';

    access_log /var/log/nginx/stream.log basic;

    server {
        listen 3306;
        proxy_pass mysql_backend;
    }
}
```

### 7. `stream` vs `http` 模块

| 功能     | `http` (Layer 7)         | `stream` (Layer 4)       |
| -------- | ------------------------ | ------------------------ |
| 代理协议 | HTTP/HTTPS               | TCP/UDP                  |
| 内容修改 | 支持（如修改 HTTP 头部） | 不支持（纯数据流）       |
| 负载均衡 | 复杂规则（如 `rewrite`） | 仅基于 IP/端口           |
| 适用场景 | Web 服务、API 网关       | 数据库、Redis、DNS、SMTP |

### 8. 适用场景

- **数据库负载均衡**：如 MySQL、PostgreSQL、Redis、MongoDB。
- **游戏服务器**：提供 TCP/UDP 代理和负载均衡。
- **DNS 代理**：转发 DNS 查询请求。
- **邮件代理**：支持 SMTP、IMAP 代理。
- **SSH 代理**：通过 TCP 代理 SSH 连接。

### 9. 总结

Nginx `stream` 模块提供了一种高效的 L4 代理解决方案，适用于需要 TCP/UDP 负载均衡的场景。虽然不具备 L7 代理的深度流量管理能力，但它的轻量级架构和高并发处理能力，使其成为网络代理和负载均衡的理想选择。