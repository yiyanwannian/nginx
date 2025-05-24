# NGINX HTTP 请求解析与多阶段处理机制

## 概述

本文是 NGINX HTTP 请求处理流程系列的第三篇，重点分析 HTTP 请求解析、多阶段处理机制和内容生成过程。这些是 NGINX 灵活性和可扩展性的核心，通过模块化的阶段处理，NGINX 能够支持各种复杂的业务逻辑。

## 5. HTTP 请求解析阶段

### 5.1 请求行解析

**源码位置**: `src/http/ngx_http_request.c` - `ngx_http_process_request_line()`

当客户端发送 HTTP 请求时，NGINX 首先解析请求行（方法、URI、协议版本）。

**功能概述**：
请求行解析是 HTTP 协议处理的第一步，负责从原始字节流中提取结构化的请求信息：
- **协议解析**：按照 HTTP 协议规范解析请求行格式（方法 URI 版本）
- **方法识别**：识别 HTTP 方法（GET、POST、PUT、DELETE 等）并进行有效性检查
- **URI 处理**：提取请求 URI，处理 URL 编码，分离路径和查询参数
- **版本检测**：识别 HTTP 协议版本（HTTP/1.0、HTTP/1.1、HTTP/2）
- **状态转换**：解析完成后将处理流程转向请求头解析阶段

这个阶段为后续的路由匹配、权限检查和内容生成奠定了基础。

```c
static void ngx_http_process_request_line(ngx_event_t *rev)
{
    ngx_int_t            rc;
    ngx_connection_t    *c;
    ngx_http_request_t  *r;

    c = rev->data;
    r = c->data;

    // 检查超时
    if (rev->timedout) {
        ngx_log_error(NGX_LOG_INFO, c->log, NGX_ETIMEDOUT, "client timed out");
        ngx_http_close_request(r, NGX_HTTP_REQUEST_TIME_OUT);
        return;
    }

    // 读取更多数据到缓冲区
    rc = ngx_http_read_request_header(r);
    if (rc != NGX_OK) {
        if (rc == NGX_AGAIN) {
            return;  // 需要更多数据
        }
        ngx_http_close_request(r, NGX_HTTP_BAD_REQUEST);
        return;
    }

    // 解析请求行：GET /path HTTP/1.1
    rc = ngx_http_parse_request_line(r, r->header_in);

    if (rc == NGX_OK) {
        // 请求行解析完成，处理 URI
        if (ngx_http_process_request_uri(r) != NGX_OK) {
            return;
        }

        // 设置下一个处理器：解析请求头
        rev->handler = ngx_http_process_request_headers;
        ngx_http_process_request_headers(rev);
        return;

    } else if (rc == NGX_AGAIN) {
        return;  // 需要更多数据
    }

    // 解析错误
    ngx_http_close_request(r, NGX_HTTP_BAD_REQUEST);
}
```

**关键功能**:
- 解析 HTTP 方法（GET、POST、PUT 等）
- 提取请求 URI 和查询参数
- 识别 HTTP 协议版本
- 处理 URI 编码和规范化

### 5.2 请求头解析

**源码位置**: `src/http/ngx_http_request.c` - `ngx_http_process_request_headers()`

请求行解析完成后，NGINX 解析 HTTP 请求头。

**功能概述**：
请求头解析是 HTTP 协议处理的关键环节，负责提取和处理客户端发送的各种元数据：
- **头部解析**：逐行解析 HTTP 请求头，处理键值对格式
- **特殊头部处理**：对关键头部（Host、Content-Length、Connection 等）进行特殊处理
- **虚拟主机选择**：根据 Host 头部选择对应的服务器配置
- **内容协商**：处理 Accept、Accept-Encoding 等协商头部
- **连接管理**：根据 Connection 头部决定连接保持策略

这个阶段收集的信息将影响后续的路由选择、内容生成和响应格式。

```c
static void ngx_http_process_request_headers(ngx_event_t *rev)
{
    ngx_int_t                   rc;
    ngx_table_elt_t            *h;
    ngx_connection_t           *c;
    ngx_http_header_t          *hh;
    ngx_http_request_t         *r;
    ngx_http_core_srv_conf_t   *cscf;
    ngx_http_core_main_conf_t  *cmcf;

    c = rev->data;
    r = c->data;

    // 检查超时
    if (rev->timedout) {
        ngx_log_error(NGX_LOG_INFO, c->log, NGX_ETIMEDOUT, "client timed out");
        ngx_http_close_request(r, NGX_HTTP_REQUEST_TIME_OUT);
        return;
    }

    cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);
    rc = NGX_AGAIN;

    // 循环解析所有请求头
    for ( ;; ) {
        if (rc == NGX_AGAIN) {
            // 读取更多数据
            if (ngx_http_read_request_header(r) != NGX_OK) {
                return;
            }
        }

        // 解析单个请求头行
        rc = ngx_http_parse_header_line(r, r->header_in, cscf->underscores_in_headers);

        if (rc == NGX_OK) {
            // 成功解析一个请求头
            r->request_length += r->header_in->pos - r->header_name_start;

            if (r->invalid_header && cscf->ignore_invalid_headers) {
                continue;  // 忽略无效的请求头
            }

            // 查找请求头处理器
            hh = ngx_hash_find(&cmcf->headers_in_hash, h->hash, h->lowcase_key, h->key.len);
            if (hh && hh->handler(r, h, hh->offset) != NGX_OK) {
                return;
            }

            continue;

        } else if (rc == NGX_HTTP_PARSE_HEADER_DONE) {
            // 所有请求头解析完成
            r->request_length += r->header_in->pos - r->header_name_start;
            r->http_state = NGX_HTTP_PROCESS_REQUEST_STATE;

            // 处理请求头并开始请求处理
            rc = ngx_http_process_request_header(r);
            if (rc != NGX_OK) {
                return;
            }

            ngx_http_process_request(r);  // 进入多阶段处理
            return;

        } else if (rc == NGX_AGAIN) {
            continue;  // 需要更多数据
        }

        // 解析错误
        ngx_log_error(NGX_LOG_INFO, c->log, 0, "client sent invalid header line");
        ngx_http_close_request(r, NGX_HTTP_BAD_REQUEST);
        return;
    }
}
```

**关键功能**:
- 解析各种 HTTP 请求头（Host、Content-Length、Connection 等）
- 处理特殊请求头（如 Host 用于虚拟主机选择）
- 验证请求头格式和内容
- 为后续处理准备请求上下文

### 5.3 请求对象创建

**源码位置**: `src/http/ngx_http_request.c` - `ngx_http_create_request()`

NGINX 为每个 HTTP 请求创建一个请求对象。

**功能概述**：
请求对象创建是 HTTP 处理流程的核心环节，为每个请求建立完整的处理上下文：
- **内存池分配**：为请求创建独立的内存池，便于资源管理和自动回收
- **配置绑定**：将请求与相应的主配置、服务器配置和位置配置关联
- **事件处理器设置**：配置读写事件处理函数，支持异步处理
- **数据结构初始化**：初始化请求头、响应头、变量等核心数据结构
- **模块上下文准备**：为各个 HTTP 模块预留上下文空间

这个对象将贯穿整个请求处理生命周期，承载所有相关的状态和数据。

```c
ngx_http_request_t *ngx_http_create_request(ngx_connection_t *c)
{
    ngx_http_request_t        *r;
    ngx_http_log_ctx_t        *ctx;
    ngx_http_connection_t     *hc;

    // 分配请求对象内存池
    r = ngx_http_alloc_request(c);
    if (r == NULL) {
        return NULL;
    }

    hc = c->data;

    // 初始化请求基本信息
    r->http_connection = hc;
    r->signature = NGX_HTTP_MODULE;
    r->connection = c;
    r->main_conf = hc->conf_ctx->main_conf;    // 主配置
    r->srv_conf = hc->conf_ctx->srv_conf;      // 服务器配置
    r->loc_conf = hc->conf_ctx->loc_conf;      // 位置配置

    // 设置读写事件处理器
    r->read_event_handler = ngx_http_block_reading;      // 阻塞读取
    r->write_event_handler = ngx_http_core_run_phases;   // 阶段处理引擎

    // 初始化响应头表
    if (ngx_list_init(&r->headers_out.headers, r->pool, 20, sizeof(ngx_table_elt_t)) != NGX_OK) {
        ngx_destroy_pool(r->pool);
        return NULL;
    }

    // 初始化模块上下文数组
    r->ctx = ngx_pcalloc(r->pool, sizeof(void *) * ngx_http_max_module);
    if (r->ctx == NULL) {
        ngx_destroy_pool(r->pool);
        return NULL;
    }

    // 设置日志上下文
    ctx = c->log->data;
    ctx->request = r;
    ctx->current_request = r;
    r->log_handler = ngx_http_log_error_handler;

    return r;
}
```

## 6. 多阶段处理机制

### 6.1 处理阶段概览

NGINX 将 HTTP 请求处理分为 11 个阶段，每个阶段可以注册多个处理器。

**功能概述**：
多阶段处理机制是 NGINX 模块化架构的核心，将复杂的 HTTP 请求处理分解为有序的处理阶段：
- **模块化设计**：每个功能模块可以注册到特定阶段，实现功能解耦
- **有序执行**：阶段按固定顺序执行，确保处理逻辑的一致性
- **灵活扩展**：新模块可以轻松集成到现有的处理流程中
- **性能优化**：只有需要的阶段才会被执行，避免不必要的处理开销

这种设计使得 NGINX 既保持了高性能，又具备了极强的可扩展性。

```c
// 源码位置: src/http/ngx_http_core_module.h
typedef enum {
    NGX_HTTP_POST_READ_PHASE = 0,      // 读取请求内容阶段
    NGX_HTTP_SERVER_REWRITE_PHASE,     // Server 级别重写阶段
    NGX_HTTP_FIND_CONFIG_PHASE,        // 配置查找阶段
    NGX_HTTP_REWRITE_PHASE,            // Location 级别重写阶段
    NGX_HTTP_POST_REWRITE_PHASE,       // 重写后处理阶段
    NGX_HTTP_PREACCESS_PHASE,          // 访问权限检查预处理阶段
    NGX_HTTP_ACCESS_PHASE,             // 访问权限检查阶段
    NGX_HTTP_POST_ACCESS_PHASE,        // 访问权限检查后处理阶段
    NGX_HTTP_PRECONTENT_PHASE,         // 生成内容前处理阶段
    NGX_HTTP_CONTENT_PHASE,            // 内容产生阶段
    NGX_HTTP_LOG_PHASE                 // 日志记录阶段
} ngx_http_phases;
```

### 6.2 阶段处理引擎

**源码位置**: `src/http/ngx_http_core_module.c` - `ngx_http_core_run_phases()`

阶段处理引擎是 NGINX 模块化架构的核心。

**功能概述**：
阶段处理引擎是 NGINX 请求处理的调度中心，负责协调各个处理阶段的执行：
- **顺序调度**：按照预定义的阶段顺序依次调用各个处理器
- **返回值处理**：根据处理器的返回值决定后续的执行流程
- **异步支持**：支持异步处理，允许处理器挂起请求等待外部事件
- **错误处理**：统一处理各种错误情况和特殊响应码
- **性能优化**：通过快速路径和跳转机制提高处理效率

这个引擎确保了请求处理的有序性和可靠性，是 NGINX 高性能的关键组件。

```c
void ngx_http_core_run_phases(ngx_http_request_t *r)
{
    ngx_int_t                   rc;
    ngx_http_phase_handler_t   *ph;
    ngx_http_core_main_conf_t  *cmcf;

    cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);
    ph = cmcf->phase_engine.handlers;  // 获取阶段处理器数组

    // 循环执行各个阶段处理器
    while (ph[r->phase_handler].checker) {
        // 调用阶段检查器
        rc = ph[r->phase_handler].checker(r, &ph[r->phase_handler]);

        if (rc == NGX_OK) {
            return;  // 请求处理完成
        }

        if (rc == NGX_DECLINED) {
            r->phase_handler++;  // 继续下一个处理器
            continue;
        }

        if (rc == NGX_AGAIN || rc == NGX_DONE) {
            return;  // 等待更多数据或异步处理
        }

        // 处理错误或特殊响应码
        ngx_http_finalize_request(r, rc);
        return;
    }
}
```

### 6.3 关键处理阶段详解

#### 6.3.1 FIND_CONFIG 阶段 - 配置查找

**源码位置**: `src/http/ngx_http_core_module.c` - `ngx_http_core_find_config_phase()`

**功能概述**：
配置查找阶段是请求处理的关键环节，负责根据请求 URI 找到匹配的 location 配置：
- **Location 匹配**：根据 URI 路径匹配最合适的 location 配置块
- **配置更新**：将请求对象与匹配的 location 配置关联
- **权限检查**：验证外部请求是否可以访问内部 location
- **大小限制**：检查请求体大小是否超过配置的限制
- **重定向处理**：处理需要重定向的特殊情况

这个阶段确定了请求的最终处理配置，影响后续所有阶段的行为。

```c
ngx_int_t ngx_http_core_find_config_phase(ngx_http_request_t *r, ngx_http_phase_handler_t *ph)
{
    ngx_int_t                  rc;
    ngx_http_core_loc_conf_t  *clcf;

    r->content_handler = NULL;
    r->uri_changed = 0;

    // 查找匹配的 location 配置
    rc = ngx_http_core_find_location(r);
    if (rc == NGX_ERROR) {
        ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
        return NGX_OK;
    }

    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

    // 检查内部 location 访问权限
    if (!r->internal && clcf->internal) {
        ngx_http_finalize_request(r, NGX_HTTP_NOT_FOUND);
        return NGX_OK;
    }

    // 更新请求的 location 配置
    ngx_http_update_location_config(r);

    // 检查请求体大小限制
    if (r->headers_in.content_length_n != -1
        && !r->discard_body
        && clcf->client_max_body_size
        && clcf->client_max_body_size < r->headers_in.content_length_n)
    {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "client intended to send too large body: %O bytes",
                      r->headers_in.content_length_n);

        (void) ngx_http_discard_request_body(r);
        ngx_http_finalize_request(r, NGX_HTTP_REQUEST_ENTITY_TOO_LARGE);
        return NGX_OK;
    }

    // 处理重定向情况
    if (rc == NGX_DONE) {
        r->headers_out.location = ngx_list_push(&r->headers_out.headers);
        if (r->headers_out.location == NULL) {
            ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
            return NGX_OK;
        }

        r->headers_out.location->hash = 1;
        ngx_str_set(&r->headers_out.location->key, "Location");
        r->headers_out.location->value = clcf->name;

        ngx_http_finalize_request(r, NGX_HTTP_MOVED_PERMANENTLY);
        return NGX_OK;
    }

    r->phase_handler++;
    return NGX_AGAIN;
}
```

#### 6.3.2 ACCESS 阶段 - 访问控制

**源码位置**: `src/http/ngx_http_core_module.c` - `ngx_http_core_access_phase()`

**功能概述**：
访问控制阶段是安全防护的核心环节，负责验证客户端是否有权限访问请求的资源：
- **多层验证**：支持多个访问控制模块同时工作，如 IP 限制、用户认证等
- **灵活策略**：可以配置 "满足任一条件" 或 "满足所有条件" 的访问策略
- **认证支持**：集成基本认证、摘要认证等多种认证机制
- **异步处理**：支持异步的访问控制检查，如外部认证服务
- **错误累积**：收集所有访问控制模块的结果，统一决定最终的访问权限

这个阶段确保了系统的安全性，防止未授权访问。

```c
ngx_int_t ngx_http_core_access_phase(ngx_http_request_t *r, ngx_http_phase_handler_t *ph)
{
    ngx_int_t  rc;

    // 调用访问控制处理器
    rc = ph->handler(r);

    if (rc == NGX_DECLINED) {
        r->phase_handler++;
        return NGX_AGAIN;  // 继续下一个访问控制模块
    }

    if (rc == NGX_AGAIN || rc == NGX_DONE) {
        return NGX_OK;  // 等待异步处理完成
    }

    if (rc == NGX_OK) {
        // 访问控制通过
        r->access_code = 0;

        if (r->headers_out.www_authenticate) {
            r->headers_out.www_authenticate->hash = 0;  // 清除认证头
        }

        r->phase_handler++;
        return NGX_AGAIN;
    }

    if (rc == NGX_HTTP_FORBIDDEN || rc == NGX_HTTP_UNAUTHORIZED) {
        // 访问被拒绝，但继续其他检查
        if (r->access_code != rc) {
            r->access_code = rc;

            if (r->headers_out.www_authenticate) {
                r->headers_out.www_authenticate->hash = 1;  // 保留认证头
            }
        }

        r->phase_handler++;
        return NGX_AGAIN;  // 继续其他访问控制检查
    }

    // 处理错误或特殊响应
    ngx_http_finalize_request(r, rc);
    return NGX_OK;
}
```

#### 6.3.3 CONTENT 阶段 - 内容生成

**源码位置**: `src/http/ngx_http_core_module.c` - `ngx_http_core_content_phase()`

**功能概述**：
内容生成阶段是 HTTP 请求处理的核心，负责生成响应内容：
- **内容处理器选择**：根据配置和请求特征选择合适的内容处理器
- **多种内容源**：支持静态文件、动态内容、代理转发等多种内容来源
- **处理器链**：按优先级尝试多个内容处理器，直到找到能处理的
- **错误处理**：当没有合适的处理器时，返回相应的错误响应
- **性能优化**：通过内容处理器缓存和快速路径提高处理效率

这个阶段决定了最终返回给客户端的内容，是整个请求处理流程的核心。

```c
ngx_int_t ngx_http_core_content_phase(ngx_http_request_t *r, ngx_http_phase_handler_t *ph)
{
    ngx_int_t  rc;
    ngx_str_t  path;

    if (r->content_handler) {
        // 已有内容处理器，直接调用
        r->write_event_handler = ngx_http_request_empty_handler;
        ngx_http_finalize_request(r, r->content_handler(r));
        return NGX_OK;
    }

    // 尝试当前阶段的内容处理器
    rc = ph->handler(r);

    if (rc != NGX_DECLINED) {
        // 处理器接受了请求
        ngx_http_finalize_request(r, rc);
        return NGX_OK;
    }

    // 当前处理器拒绝，尝试下一个
    ph++;
    r->phase_handler++;

    if (ph->checker) {
        r->content_handler = ph->handler;
        return NGX_AGAIN;
    }

    // 没有找到合适的内容处理器
    if (r->uri.data[r->uri.len - 1] == '/') {
        // 目录请求但没有索引处理器
        if (ngx_http_map_uri_to_path(r, &path, &root, 0) != NULL) {
            ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                          "directory index of \"%s\" is forbidden", path.data);
        }
        ngx_http_finalize_request(r, NGX_HTTP_FORBIDDEN);
        return NGX_OK;
    }

    // 没有找到任何处理器
    ngx_log_error(NGX_LOG_ERR, r->connection->log, 0, "no handler found");
    ngx_http_finalize_request(r, NGX_HTTP_NOT_FOUND);
    return NGX_OK;
}
```

## 7. 模块注册机制

### 7.1 模块初始化

各个 HTTP 模块在初始化时注册到相应的处理阶段。

**功能概述**：
模块注册机制是 NGINX 扩展性的基础，允许各种功能模块集成到请求处理流程中：
- **阶段选择**：模块根据功能特点选择合适的处理阶段进行注册
- **处理器注册**：将模块的处理函数注册到指定阶段的处理器数组中
- **优先级控制**：通过注册顺序和阶段选择控制模块的执行优先级
- **动态扩展**：支持第三方模块的动态加载和注册

这种机制使得 NGINX 具备了强大的模块化扩展能力。

```c
// 示例：静态文件模块注册到 CONTENT 阶段
static ngx_int_t ngx_http_static_init(ngx_conf_t *cf)
{
    ngx_http_handler_pt        *h;
    ngx_http_core_main_conf_t  *cmcf;

    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

    // 注册到 CONTENT 阶段
    h = ngx_array_push(&cmcf->phases[NGX_HTTP_CONTENT_PHASE].handlers);
    if (h == NULL) {
        return NGX_ERROR;
    }

    *h = ngx_http_static_handler;  // 设置处理函数

    return NGX_OK;
}
```

### 7.2 常见模块的阶段分布

```c
// 不同模块注册到不同阶段的示例：

// 重写模块 - SERVER_REWRITE 和 REWRITE 阶段
NGX_HTTP_SERVER_REWRITE_PHASE: ngx_http_rewrite_handler
NGX_HTTP_REWRITE_PHASE:        ngx_http_rewrite_handler

// 访问控制模块 - ACCESS 阶段
NGX_HTTP_ACCESS_PHASE:         ngx_http_access_handler
NGX_HTTP_ACCESS_PHASE:         ngx_http_auth_basic_handler

// 限流模块 - PREACCESS 阶段
NGX_HTTP_PREACCESS_PHASE:      ngx_http_limit_req_handler
NGX_HTTP_PREACCESS_PHASE:      ngx_http_limit_conn_handler

// 内容处理模块 - CONTENT 阶段
NGX_HTTP_CONTENT_PHASE:        ngx_http_static_handler
NGX_HTTP_CONTENT_PHASE:        ngx_http_index_handler
NGX_HTTP_CONTENT_PHASE:        ngx_http_autoindex_handler
```

## 小结

本文深入分析了 NGINX HTTP 请求解析和多阶段处理机制：

1. **请求解析**: 请求行解析、请求头解析、请求对象创建
2. **多阶段处理**: 11 个处理阶段的设计和实现
3. **阶段处理引擎**: 模块化的处理器调用机制
4. **关键阶段**: 配置查找、访问控制、内容生成的详细实现
5. **模块注册**: 各模块如何注册到相应处理阶段

这种多阶段处理机制是 NGINX 灵活性和可扩展性的核心，使得各种功能模块能够有序协作，处理复杂的 HTTP 请求。在下一篇文章中，我们将分析响应生成、过滤器链和高并发优化机制。
