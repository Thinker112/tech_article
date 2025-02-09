# Nginx负载均衡策略

## 负载均衡策略

### 1. ip_hash

根据客户端 IP 地址将请求分配到后端服务器。它的作用是保证来自同一客户端 IP 的请求始终转发到同一台后端服务器，这在需要维护会话状态的应用场景中非常有用。

以下是一个使用 `ip_hash` 的简单示例：

```nginx
http {
    upstream backend {
        ip_hash;  # 启用 ip_hash 负载均衡策略

        server 192.168.1.101:80;
        server 192.168.1.102:80;
        server 192.168.1.103:80;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://backend;
        }
    }
}
```

#### 配置说明

1. **`ip_hash;`**：

   - 开启 `ip_hash` 策略。
   - 客户端的 IP 地址将通过哈希计算，始终被路由到固定的后端服务器。

2. **后端服务器**：

   - 配置在 `upstream` 中的服务器 IP 和端口。

   - 如果某些服务器需要临时下线维护，建议使用 `down` 参数，例如：

     ```nginx
     server 192.168.1.101:80 down;
     ```

#### 注意事项

1. **动态 IP 或代理的问题**：
   - 如果客户端的 IP 通过代理或 NAT 转发，可能会导致多个客户端共享一个 IP，从而不均衡。
   - 可以通过配置 `real_ip_module` 使用真实客户端 IP。
2. **不支持权重**：
   - `ip_hash` 不支持为后端服务器设置不同的权重。如果需要权重功能，可以考虑其他负载均衡算法，例如 `least_conn` 或 `random`.
3. **非对称请求**：
   - 如果后端服务器的请求量需要均衡分配，而不需要绑定客户端 IP，可以考虑不使用 `ip_hash`。

#### `ip_hash` 在 HTTPS 下的问题

1. **代理层的 IP 地址问题**

在 HTTPS 的场景中，通常会有一个终止 SSL 的代理层（如负载均衡器或 CDN），这些代理会处理 SSL/TLS 协议并转发请求到 Nginx。由于代理层的存在，Nginx 接收到的客户端 IP 地址可能是代理的 IP，而不是实际客户端的真实 IP。

**影响：**

- **`ip_hash` 依赖于客户端的真实 IP 地址**。如果 Nginx 看到的 IP 地址是代理的 IP，那么多个客户端可能会被错误地路由到同一个后端服务器。

**解决方法：**

- 在代理层启用 **真实客户端 IP 透传**（如通过 `X-Forwarded-For` 或 `X-Real-IP` 头）。
- 在 Nginx 中使用 `real_ip_module` 来恢复真实客户端 IP。

配置示例：

```nginx
http {
    set_real_ip_from 192.168.1.0/24;  # 信任的代理 IP 段
    real_ip_header X-Forwarded-For;
    real_ip_recursive on;

    upstream backend {
        ip_hash;
        server backend1.example.com;
        server backend2.example.com;
    }

    server {
        listen 443 ssl;
        location / {
            proxy_pass http://backend;
        }
    }
}
```

2. **客户端 IP 动态变化**

在 HTTPS 的场景下，客户端与服务器之间可能存在多个中间节点（如 NAT 网关、VPN 等），这些节点可能导致客户端 IP 地址在会话期间发生变化。

**影响：**

- 如果客户端的 IP 地址发生变化，`ip_hash` 无法保证路由到同一个后端服务器，从而破坏了会话保持。

**解决方法：**

- **避免使用 `ip_hash` 方法**，改用基于 Cookie 的 Sticky Session，因为 Cookie 是与客户端绑定的，不受 IP 地址变化的影响。

3. **负载均衡与 TLS 会话复用的干扰**

一些负载均衡器在处理 HTTPS 时，可能会启用 **TLS 会话复用** 来优化性能。虽然这对加密性能有帮助，但可能干扰 `ip_hash` 方法的会话保持。

**影响：**

- 复用的会话可能会被负载均衡器分配到不同的后端服务器，破坏 `ip_hash` 的一致性。

**解决方法：**

- 配置负载均衡器以支持与后端的一致性会话保持，或者使用更可靠的方法（如基于 Cookie）。

### 2. 权重

在 Nginx 中，通过设置后端服务器的 **权重（weight）**，可以实现基于负载能力的流量分配。权重值越大，分配到该服务器的请求就越多。

#### 配置方法

    http {
        upstream backend {
            # 配置权重
            server 192.168.1.101:80 weight=3;  # 权重为 3
            server 192.168.1.102:80 weight=2;  # 权重为 2
            server 192.168.1.103:80 weight=1;  # 权重为 1
        }
    
        server {
            listen 80;
    
            location / {
                proxy_pass http://backend;
            }
        }
    }

1. **`weight=N`**:
   - `N` 表示权重值（默认为 `1`）。
   - 权重值越大，服务器接收的请求比例越高。
2. **分配比例**:
   - 根据上述配置，3 台服务器的请求分配比例约为 `3:2:1`。
   - 即，每 6 个请求中，`192.168.1.101` 处理 3 个，`192.168.1.102` 处理 2 个，`192.168.1.103` 处理 1 个。
3. **无权重时的默认行为**:
   - 如果不设置 `weight`，所有服务器默认的权重为 `1`，请求被均匀分配。

#### 适用场景

- **性能不均衡的服务器**：当后端服务器的硬件配置、网络带宽或处理能力不同，可以通过权重进行流量分配优化。
- **渐进式上线**：在新增服务器时，可以先为新服务器设置较低的权重，逐步增加以观察其表现。

#### 注意事项

1. **权重与其他策略的兼容性**：
   - 如果同时启用了其他负载均衡策略（如 `ip_hash`），则权重会被忽略。
   - 权重仅适用于默认的轮询（round-robin）策略。
2. **健康检查**：
   - 可以结合 `nginx_upstream_check_module` 或其他健康检查模块，确保权重分配只应用于健康的服务器。

### 3. least_conn

将新请求分配给当前活动连接数最少的后端服务器。这种策略适用于请求处理时间差异较大或需要更均衡连接分配的场景。

#### 配置方法

以下是使用 `least_conn` 策略的一个示例：

```nginx
http {
    upstream backend {
        least_conn;  # 启用 least_conn 策略

        server 192.168.1.101:80;
        server 192.168.1.102:80;
        server 192.168.1.103:80;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://backend;
        }
    }
}
```

#### 配置说明

1. **`least_conn;`**:

   - 开启 `least_conn` 策略。
   - 每次请求会转发到当前活动连接数最少的服务器。

2. **后端服务器**:

   - 服务器列表中可以设置其他参数（如权重），以进一步优化流量分配。例如：

     ```nginx
     upstream backend {
         least_conn;
         server 192.168.1.101:80 weight=3;
         server 192.168.1.102:80 weight=2;
         server 192.168.1.103:80 weight=1;
     }
     ```

   - 此时，权重会与 `least_conn` 策略结合使用，权重越高的服务器，即使活动连接数较多，也可能更早接收新请求。

#### 适用场景

- **长时间处理的请求**：对于需要长连接或请求处理时间不均匀的应用（如文件上传/下载、实时通信），`least_conn` 能有效均衡服务器负载。
- **动态请求负载**：当后端服务器的负载可能随时间变化，`least_conn` 可以更灵活地调整请求分配。

#### 注意事项

1. **健康检查**：
   - 配置健康检查模块（如 `ngx_http_upstream_check_module`），以避免向故障服务器分发请求。
2. **空闲服务器问题**：
   - 在初始阶段（所有服务器都没有连接时），`least_conn` 会以轮询方式分配请求。
3. **结合其他策略**：
   - 如果需要与其他策略结合（如权重、会话保持），需要仔细设计以确保逻辑合理。

#### 与其他策略的比较

| 策略          | 描述                                       | 适用场景                                 |
| ------------- | ------------------------------------------ | ---------------------------------------- |
| `round-robin` | 默认策略，按顺序轮询分发请求               | 服务器性能均衡、请求处理时间相近的场景   |
| `least_conn`  | 优先分配给当前活动连接数最少的服务器       | 请求处理时间不均匀、长连接等场景         |
| `ip_hash`     | 根据客户端 IP 地址固定请求分配到某台服务器 | 需要会话保持（如购物车、用户认证）的场景 |

### 4. fair

`fair` 是一种非官方的负载均衡策略，提供了更智能的请求分配机制。与默认的轮询（round-robin）或 `least_conn` 不同，`fair` 策略会根据后端服务器的响应时间动态分配请求，优先将新请求发送到响应时间较短的服务器。

#### 配置方法

`fair` 策略并未内置在 Nginx 的官方模块中，需要通过第三方模块 **`nginx-upstream-fair`** 来实现。

#### 配置 `fair` 策略

在安装并启用模块后，可以在 Nginx 配置文件中使用 `fair` 策略：

```nginx
http {
    upstream backend {
        fair;  # 启用 fair 策略

        server 192.168.1.101:80;
        server 192.168.1.102:80;
        server 192.168.1.103:80;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://backend;
        }
    }
}
```

#### 配置说明

1. **`fair`**:
   - 启用 `fair` 策略，根据后端服务器的响应时间分配请求。
   - 请求优先发送给响应时间短的服务器，从而提升整体性能。
2. **后端服务器**:
   - 和其他负载均衡策略一样，可以定义后端服务器列表。

#### 适用场景

- **响应时间差异较大的服务器**： 后端服务器的硬件性能或负载情况不同，`fair` 策略可以自动调整流量分配。
- **动态负载场景**： 在某些高并发场景中，不同时间段服务器的响应能力可能波动，`fair` 策略能更灵活地适配。

### 5. uri_hash

**`uri_hash`** 是一种非官方的负载均衡策略，它会根据请求的 URI 进行哈希计算，并将请求分配到后端服务器。这种策略适用于特定场景，比如缓存服务器的负载均衡，能够保证相同的 URI 请求始终分配到同一台后端服务器，以提高缓存命中率。

#### 配置方法

`uri_hash` 不属于 Nginx 官方的内置策略，需要使用第三方模块 **`ngx_http_upstream_hash`** 实现。

1. **下载模块**：

   - 从 [第三方模块库](https://github.com/yaoweibin/nginx-upstream-consistent-hash) 获取模块源代码。

2. **编译模块**： 重新编译 Nginx，并添加 `ngx_http_upstream_hash` 模块：

   ```bash
   ./configure --add-module=/path/to/nginx-upstream-hash
   make
   make install
   ```

3. **验证安装**： 检查是否支持 `hash` 指令。

#### 配置 `uri_hash`

在启用模块后，可以使用以下配置：

```nginx
http {
    upstream backend {
        hash $request_uri consistent;  # 使用 URI 作为哈希的依据，启用一致性哈希

        server 192.168.1.101:80;
        server 192.168.1.102:80;
        server 192.168.1.103:80;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://backend;
        }
    }
}
```

#### 配置说明

1. **`hash`**:
   - 使用 `$request_uri` 作为哈希的依据。
   - 也可以根据需要选择其他变量，如 `$uri`（不含查询字符串）或 `$request_uri`（含查询字符串）。
2. **`consistent`**:
   - 启用一致性哈希算法（Consistent Hashing），可以在服务器增删时最小化缓存重新分配的影响。
3. **后端服务器**:
   - 配置后端服务器列表，与其他策略一致。

#### 适用场景

1. **缓存服务**：
   - 对于基于 URI 的缓存系统（如图片或文件缓存），`uri_hash` 可以提升缓存命中率。
2. **请求绑定**：
   - 某些应用需要根据 URI 的特性将请求固定到特定服务器。
3. **分布式存储**：
   - 在分布式文件存储系统中，根据 URI 将资源分布到不同的存储节点。

#### 注意事项

1. **模块依赖性**：
   - `ngx_http_upstream_hash` 是第三方模块，可能需要额外的维护工作。
2. **健康检查**：
   - 如果某台后端服务器出现故障，未配合健康检查模块时，相关 URI 请求可能无法处理。
3. **负载分布**：
   - 哈希算法可能会导致流量分布不均，需结合实际负载情况进行评估。
4. **动态调整**：
   - 如果需要频繁增删后端服务器，建议启用 `consistent` 参数，以减小影响。

#### 替代方案

如果不想使用第三方模块，可以通过 Nginx 的 Lua 模块（如 OpenResty）自定义类似的哈希逻辑。例如，利用 `$uri` 变量编写负载均衡规则。

## 会话保持

### 1. 使用 Sticky Sessions 第三方模块

如果使用开源版 Nginx，可以通过编译支持 `nginx-sticky-module-ng`。

#### 配置示例：

```
http {
    upstream backend {
        sticky;
        server backend1.example.com;
        server backend2.example.com;
    }

    server {
        listen 80;
        location / {
            proxy_pass http://backend;
        }
    }
}
```

### 2. 使用 Nginx 自定义变量

可以通过手动创建和管理 Cookie 的方式来实现。

#### 配置示例：

```
nginx复制编辑http {
    upstream backend {
        server backend1.example.com;
        server backend2.example.com;
    }

    server {
        listen 80;

        location / {
            # 检查是否有 sticky_cookie
            set $sticky_cookie $cookie_srv_id;

            # 如果没有，设置一个随机后端
            if ($sticky_cookie = "") {
                set $sticky_cookie "backend1";  # 简单示例，可以改为动态生成
                add_header Set-Cookie "srv_id=$sticky_cookie; Path=/";
            }

            proxy_pass http://$sticky_cookie;
        }
    }
}
```

### 3. 注意事项

1. 如果使用基于 IP 的方法（`ip_hash`），在启用 HTTPS 时，可能会影响客户端会话保持。
2. 在配置会话保持时，应考虑后端服务器是否能够支持跨服务器共享会话（如使用 Redis 或数据库同步会话数据）。

## HTTP-Keepalive

```nginx
http{
    keepalive_time 60; #限制最大连接时长，超过设定时间后TCP连接将强制关闭。默认时间：1h
    keepalive_timeout 65;#keepalive空闲时间，超出空闲时间keepalive将失效。
    send_timeout 30s;#控制服务器向客户端发送响应数据时的超时时间, 默认值：60s
    keepalive_request 1000;#一个TCP复用中 可以并发接收的请求个数。
}
```

