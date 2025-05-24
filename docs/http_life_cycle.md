# HTTP 请求在 NGINX 中的生命周期

本文档详细分析了 HTTP 请求在 NGINX 中的完整生命周期，从连接建立到请求处理再到响应发送，并标记了相关源代码的位置。

## 1. 连接建立阶段

当客户端与 NGINX 建立 TCP 连接时，NGINX 的事件模块会处理这个新连接。

### 1.1 连接初始化

**源码位置**: `src/http/ngx_http.c` 中的 `ngx_http_add_listening` 函数

```c
static ngx_listening_t *
ngx_http_add_listening(ngx_conf_t *cf, ngx_http_conf_addr_t *addr)
{
    // ...
    ls->handler = ngx_http_init_connection;
    // ...
}
```

当新连接到达时，`ngx_http_init_connection` 函数会被调用，它负责初始化 HTTP 连接。

### 1.2 等待请求

**源码位置**: `src/http/ngx_http_request.c` 中的 `ngx_http_wait_request_handler` 函数

```c
static void
ngx_http_wait_request_handler(ngx_event_t *rev)
{
    // ...
    c->data = ngx_http_create_request(c);
    // ...
    rev->handler = ngx_http_process_request_line;
    ngx_http_process_request_line(rev);
}
```

NGINX 等待客户端发送请求数据，一旦接收到数据，就会创建请求对象并开始处理请求行。

## 2. 请求解析阶段

### 2.1 解析请求行

**源码位置**: `src/http/ngx_http_request.c` 中的 `ngx_http_process_request_line` 函数

```c
static void
ngx_http_process_request_line(ngx_event_t *rev)
{
    // ...
    rc = ngx_http_parse_request_line(r, r->header_in);
    // ...
    rc = ngx_http_process_request_uri(r);
    // ...
    rev->handler = ngx_http_process_request_headers;
    ngx_http_process_request_headers(rev);
}
```

NGINX 解析 HTTP 请求行（如 `GET /index.html HTTP/1.1`），提取请求方法、URI 和 HTTP 版本。

### 2.2 解析请求头

**源码位置**: `src/http/ngx_http_request.c` 中的 `ngx_http_process_request_headers` 函数

```c
static void
ngx_http_process_request_headers(ngx_event_t *rev)
{
    // ...
    for ( ;; ) {
        // 读取并解析请求头
        rc = ngx_http_parse_header_line(r, r->header_in, 1);
        // ...
    }
    // ...
    rc = ngx_http_process_request_header(r);
    // ...
    ngx_http_process_request(r);
}
```

NGINX 逐行解析 HTTP 请求头，如 `Host`、`User-Agent` 等，并将它们存储在请求对象中。

### 2.3 处理请求头

**源码位置**: `src/http/ngx_http_request.c` 中的 `ngx_http_process_request_header` 函数

```c
ngx_int_t
ngx_http_process_request_header(ngx_http_request_t *r)
{
    // ...
    // 处理 Host 头
    // 处理 Connection 头
    // 处理 Content-Length 头
    // ...
}
```

NGINX 处理特殊的请求头，如 `Host`（用于虚拟主机）、`Connection`（决定是否保持连接）等。

## 3. 请求处理阶段

### 3.1 初始化请求处理

**源码位置**: `src/http/ngx_http_request.c` 中的 `ngx_http_process_request` 函数

```c
void
ngx_http_process_request(ngx_http_request_t *r)
{
    // ...
    // 处理 SSL 相关
    // 设置请求处理上下文
    // ...
    ngx_http_handler(r);
}
```

NGINX 完成请求的初始化工作，然后调用 `ngx_http_handler` 开始处理请求。

### 3.2 请求处理器

**源码位置**: `src/http/ngx_http_core_module.c` 中的 `ngx_http_handler` 函数

```c
void
ngx_http_handler(ngx_http_request_t *r)
{
    // ...
    r->phase_handler = 0;
    // ...
    r->write_event_handler = ngx_http_core_run_phases;
    ngx_http_core_run_phases(r);
}
```

`ngx_http_handler` 设置请求的阶段处理器，并开始运行请求处理阶段。

### 3.3 请求处理阶段

**源码位置**: `src/http/ngx_http_core_module.c` 中的 `ngx_http_core_run_phases` 函数和 `src/http/ngx_http_core_module.h` 中的 `ngx_http_phases` 枚举

```c
void
ngx_http_core_run_phases(ngx_http_request_t *r)
{
    ngx_int_t                   rc;
    ngx_http_phase_handler_t   *ph;
    ngx_http_core_main_conf_t  *cmcf;

    cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);
    ph = cmcf->phase_engine.handlers;

    while (ph[r->phase_handler].checker) {
        rc = ph[r->phase_handler].checker(r, &ph[r->phase_handler]);
        if (rc == NGX_OK) {
            return;
        }
    }
}
```

NGINX 定义了多个请求处理阶段，每个阶段有特定的功能：

```c
typedef enum {
    NGX_HTTP_POST_READ_PHASE = 0,
    NGX_HTTP_SERVER_REWRITE_PHASE,
    NGX_HTTP_FIND_CONFIG_PHASE,
    NGX_HTTP_REWRITE_PHASE,
    NGX_HTTP_POST_REWRITE_PHASE,
    NGX_HTTP_PREACCESS_PHASE,
    NGX_HTTP_ACCESS_PHASE,
    NGX_HTTP_POST_ACCESS_PHASE,
    NGX_HTTP_PRECONTENT_PHASE,
    NGX_HTTP_CONTENT_PHASE,
    NGX_HTTP_LOG_PHASE
} ngx_http_phases;
```

#### 3.3.1 POST_READ 阶段

在读取完请求头后立即执行，可以修改请求的属性。

#### 3.3.2 SERVER_REWRITE 阶段

在 server 配置级别执行 URL 重写规则。

#### 3.3.3 FIND_CONFIG 阶段

**源码位置**: `src/http/ngx_http_core_module.c` 中的 `ngx_http_core_find_config_phase` 函数

```c
ngx_int_t
ngx_http_core_find_config_phase(ngx_http_request_t *r, ngx_http_phase_handler_t *ph)
{
    // ...
    // 根据 URI 查找匹配的 location 配置
    // ...
}
```

根据请求的 URI 查找匹配的 location 配置。

#### 3.3.4 REWRITE 阶段

在 location 配置级别执行 URL 重写规则。

#### 3.3.5 POST_REWRITE 阶段

如果在 REWRITE 阶段修改了 URI，则可能需要重新查找 location 配置。

#### 3.3.6 PREACCESS 阶段

在访问控制前执行的阶段，如限制请求速率。

#### 3.3.7 ACCESS 阶段

**源码位置**: `src/http/ngx_http_core_module.c` 中的 `ngx_http_core_access_phase` 函数

```c
ngx_int_t
ngx_http_core_access_phase(ngx_http_request_t *r, ngx_http_phase_handler_t *ph)
{
    // ...
    // 执行访问控制模块
    // ...
}
```

执行访问控制，如基于 IP 的访问控制、基于用户名密码的认证等。

#### 3.3.8 POST_ACCESS 阶段

在访问控制后执行的阶段。

#### 3.3.9 PRECONTENT 阶段

在生成内容前执行的阶段。

#### 3.3.10 CONTENT 阶段

**源码位置**: `src/http/ngx_http_core_module.c` 中的 `ngx_http_core_content_phase` 函数

```c
ngx_int_t
ngx_http_core_content_phase(ngx_http_request_t *r, ngx_http_phase_handler_t *ph)
{
    // ...
    if (r->content_handler) {
        r->write_event_handler = ngx_http_request_empty_handler;
        ngx_http_finalize_request(r, r->content_handler(r));
        return NGX_OK;
    }
    // ...
}
```

生成响应内容的阶段，如静态文件服务、代理请求、FastCGI 处理等。

#### 3.3.11 LOG 阶段

记录请求日志的阶段。

## 4. 响应生成和发送阶段

### 4.1 响应头过滤

**源码位置**: `src/http/ngx_http_header_filter_module.c` 中的 `ngx_http_header_filter` 函数

```c
static ngx_int_t
ngx_http_header_filter(ngx_http_request_t *r)
{
    // ...
    // 生成响应头
    // ...
    return ngx_http_write_filter(r, &out);
}
```

NGINX 生成 HTTP 响应头，如 `Status`、`Content-Type`、`Content-Length` 等。

### 4.2 响应体过滤

**源码位置**: `src/http/ngx_http.h` 中的 `ngx_http_top_body_filter` 变量和 `src/http/ngx_http_core_module.c` 中的 `ngx_http_output_filter` 函数

```c
ngx_int_t
ngx_http_output_filter(ngx_http_request_t *r, ngx_chain_t *in)
{
    // ...
    rc = ngx_http_top_body_filter(r, in);
    // ...
    return rc;
}
```

NGINX 通过一系列过滤器处理响应体，如压缩、字符集转换等。

### 4.3 写入过滤器

**源码位置**: `src/http/ngx_http_write_filter_module.c` 中的 `ngx_http_write_filter` 函数

```c
ngx_int_t
ngx_http_write_filter(ngx_http_request_t *r, ngx_chain_t *in)
{
    // ...
    // 将响应数据写入客户端连接
    // ...
}
```

NGINX 将响应数据写入客户端连接。

## 5. 请求完成阶段

### 5.1 请求终结

**源码位置**: `src/http/ngx_http_request.c` 中的 `ngx_http_finalize_request` 函数

```c
void
ngx_http_finalize_request(ngx_http_request_t *r, ngx_int_t rc)
{
    // ...
    // 清理请求资源
    // 处理保持连接
    // ...
}
```

NGINX 完成请求处理，清理资源，并决定是否保持连接以处理下一个请求。

### 5.2 请求清理

**源码位置**: `src/http/ngx_http_request.c` 中的 `ngx_http_free_request` 函数

```c
static void
ngx_http_free_request(ngx_http_request_t *r, ngx_int_t rc)
{
    // ...
    // 执行请求清理回调
    // 释放请求内存
    // ...
}
```

NGINX 释放请求占用的资源，如内存池、文件描述符等。

## 6. 连接处理

### 6.1 保持连接

**源码位置**: `src/http/ngx_http_request.c` 中的 `ngx_http_set_keepalive` 函数

```c
static void
ngx_http_set_keepalive(ngx_http_request_t *r)
{
    // ...
    // 重置连接状态
    // 准备处理下一个请求
    // ...
}
```

如果客户端请求保持连接，NGINX 会重置连接状态，准备处理下一个请求。

### 6.2 关闭连接

**源码位置**: `src/http/ngx_http_request.c` 中的 `ngx_http_close_connection` 函数

```c
void
ngx_http_close_connection(ngx_connection_t *c)
{
    // ...
    // 关闭连接
    // ...
}
```

如果不需要保持连接，或者发生错误，NGINX 会关闭与客户端的连接。

## 总结

HTTP 请求在 NGINX 中的生命周期是一个复杂而精心设计的过程，包括连接建立、请求解析、请求处理、响应生成和发送、请求完成等多个阶段。每个阶段都有特定的功能和对应的源代码实现。

NGINX 的模块化设计和阶段处理机制使得它能够灵活地处理各种 HTTP 请求，并支持丰富的功能扩展。理解这个生命周期对于开发 NGINX 模块和调试 NGINX 问题非常重要。
