# NGINX 响应过滤器实现详解

## 概述

本文是 NGINX 响应过滤器系列的第二篇，详细分析常用响应过滤器的具体实现。通过深入了解这些过滤器的工作原理，可以更好地理解 NGINX 的响应处理机制，并为开发自定义过滤器提供参考。

## 1. Gzip 压缩过滤器

### 1.1 Gzip 头部过滤器

**源码位置**: `src/http/modules/ngx_http_gzip_filter_module.c` - `ngx_http_gzip_header_filter()`

Gzip 头部过滤器负责决定是否对响应进行压缩，并设置相应的响应头。

**功能概述**：
Gzip 头部过滤器是内容压缩的决策中心，负责判断响应是否适合压缩：
- **压缩条件检查**：检查内容类型、大小、编码等是否满足压缩条件
- **客户端支持检测**：通过 Accept-Encoding 头部检查客户端是否支持 gzip
- **响应头设置**：设置 Content-Encoding、Vary 等压缩相关头部
- **上下文初始化**：为后续的内容压缩准备压缩上下文
- **性能优化**：避免对不适合压缩的内容进行处理

这个过滤器的决策直接影响响应的传输效率和用户体验。

```c
static ngx_int_t ngx_http_gzip_header_filter(ngx_http_request_t *r)
{
    ngx_table_elt_t       *h;
    ngx_http_gzip_ctx_t   *ctx;
    ngx_http_gzip_conf_t  *conf;

    conf = ngx_http_get_module_loc_conf(r, ngx_http_gzip_filter_module);

    // 检查压缩条件
    if (!conf->enable
        || (r->headers_out.status != NGX_HTTP_OK
            && r->headers_out.status != NGX_HTTP_FORBIDDEN
            && r->headers_out.status != NGX_HTTP_NOT_FOUND)
        || (r->headers_out.content_encoding && r->headers_out.content_encoding->value.len)
        || (r->headers_out.content_length_n != -1 && r->headers_out.content_length_n < conf->min_length)
        || ngx_http_test_content_type(r, &conf->types) == NULL
        || r->header_only)
    {
        return ngx_http_next_header_filter(r);  // 不压缩，调用下一个过滤器
    }

    // 检查客户端是否支持 gzip
    if (ngx_http_gzip_ok(r) != NGX_OK) {
        return ngx_http_next_header_filter(r);
    }

    // 创建压缩上下文
    ctx = ngx_pcalloc(r->pool, sizeof(ngx_http_gzip_ctx_t));
    if (ctx == NULL) {
        return NGX_ERROR;
    }

    ngx_http_set_ctx(r, ctx, ngx_http_gzip_filter_module);

    // 设置 Content-Encoding 头部
    h = ngx_list_push(&r->headers_out.headers);
    if (h == NULL) {
        return NGX_ERROR;
    }

    h->hash = 1;
    h->next = NULL;
    ngx_str_set(&h->key, "Content-Encoding");
    ngx_str_set(&h->value, "gzip");
    r->headers_out.content_encoding = h;

    // 设置 Vary 头部
    r->gzip_vary = 1;

    // 清除 Content-Length（压缩后长度会变化）
    ngx_http_clear_content_length(r);
    ngx_http_clear_accept_ranges(r);
    ngx_http_weak_etag(r);

    return ngx_http_next_header_filter(r);
}
```

### 1.2 Gzip 内容过滤器

**源码位置**: `src/http/modules/ngx_http_gzip_filter_module.c` - `ngx_http_gzip_body_filter()`

Gzip 内容过滤器负责对响应体进行实际的压缩处理。

**功能概述**：
Gzip 内容过滤器是压缩处理的核心，负责对响应数据进行高效压缩：
- **流式压缩**：支持对大文件进行流式压缩，避免内存占用过大
- **压缩级别控制**：根据配置调整压缩级别，平衡压缩率和 CPU 使用
- **缓冲区管理**：高效管理压缩输入和输出缓冲区
- **错误处理**：处理压缩过程中的各种异常情况
- **性能优化**：通过缓存和批处理提高压缩效率

这个过滤器直接影响响应的传输大小和服务器的 CPU 使用率。

```c
static ngx_int_t ngx_http_gzip_body_filter(ngx_http_request_t *r, ngx_chain_t *in)
{
    int                   rc;
    ngx_uint_t            flush;
    ngx_chain_t          *cl;
    ngx_http_gzip_ctx_t  *ctx;

    ctx = ngx_http_get_module_ctx(r, ngx_http_gzip_filter_module);

    if (ctx == NULL || ctx->done || r->header_only) {
        return ngx_http_next_body_filter(r, in);  // 不需要压缩
    }

    // 初始化压缩器
    if (ctx->zstream.state == NULL) {
        if (ngx_http_gzip_filter_deflate_start(r, ctx) != NGX_OK) {
            return NGX_ERROR;
        }
    }

    // 处理输入数据链
    for (cl = in; cl; cl = cl->next) {

        // 设置压缩输入
        ctx->zstream.next_in = cl->buf->pos;
        ctx->zstream.avail_in = cl->buf->last - cl->buf->pos;

        while (ctx->zstream.avail_in > 0) {

            // 获取输出缓冲区
            if (ctx->out_buf == NULL) {
                ctx->out_buf = ngx_http_gzip_filter_get_buf(r, ctx);
                if (ctx->out_buf == NULL) {
                    return NGX_ERROR;
                }
            }

            ctx->zstream.next_out = ctx->out_buf->last;
            ctx->zstream.avail_out = ctx->out_buf->end - ctx->out_buf->last;

            // 执行压缩
            rc = deflate(&ctx->zstream, Z_NO_FLUSH);

            if (rc != Z_OK) {
                ngx_log_error(NGX_LOG_ALERT, r->connection->log, 0, "deflate() failed: %d", rc);
                return NGX_ERROR;
            }

            ctx->out_buf->last = ctx->zstream.next_out;

            // 如果输出缓冲区满了，发送数据
            if (ctx->zstream.avail_out == 0) {
                if (ngx_http_gzip_filter_add_data(r, ctx) != NGX_OK) {
                    return NGX_ERROR;
                }
            }
        }

        // 更新输入缓冲区位置
        cl->buf->pos = ctx->zstream.next_in;

        if (cl->buf->last_buf) {
            ctx->last = 1;
        }
    }

    // 如果是最后一块数据，完成压缩
    if (ctx->last) {
        if (ngx_http_gzip_filter_deflate_end(r, ctx) != NGX_OK) {
            return NGX_ERROR;
        }
        ctx->done = 1;
    }

    return ngx_http_next_body_filter(r, ctx->out);
}
```

## 2. Headers 过滤器

### 2.1 Headers 过滤器实现

**源码位置**: `src/http/modules/ngx_http_headers_filter_module.c` - `ngx_http_headers_filter()`

Headers 过滤器负责添加、修改或删除 HTTP 响应头。

**功能概述**：
Headers 过滤器是响应头管理的核心组件，提供灵活的头部操作功能：
- **自定义头部添加**：根据配置添加自定义的响应头
- **条件头部设置**：根据请求特征有条件地设置头部
- **安全头部**：添加安全相关的响应头，如 HSTS、CSP 等
- **缓存控制**：设置 Expires、Cache-Control 等缓存相关头部
- **CORS 支持**：设置跨域资源共享相关的头部

这个过滤器为 NGINX 提供了强大的响应头定制能力。

```c
static ngx_int_t ngx_http_headers_filter(ngx_http_request_t *r)
{
    ngx_str_t                 value;
    ngx_uint_t                i, safe_status;
    ngx_http_header_val_t    *h;
    ngx_http_headers_conf_t  *conf;

    if (r != r->main) {
        return ngx_http_next_header_filter(r);  // 只处理主请求
    }

    conf = ngx_http_get_module_loc_conf(r, ngx_http_headers_filter_module);

    // 检查是否有配置的头部
    if (conf->expires == NGX_HTTP_EXPIRES_OFF && conf->headers == NULL) {
        return ngx_http_next_header_filter(r);
    }

    // 检查状态码是否安全
    switch (r->headers_out.status) {
    case NGX_HTTP_OK:
    case NGX_HTTP_CREATED:
    case NGX_HTTP_NO_CONTENT:
    case NGX_HTTP_PARTIAL_CONTENT:
    case NGX_HTTP_MOVED_PERMANENTLY:
    case NGX_HTTP_MOVED_TEMPORARILY:
    case NGX_HTTP_SEE_OTHER:
    case NGX_HTTP_NOT_MODIFIED:
    case NGX_HTTP_TEMPORARY_REDIRECT:
    case NGX_HTTP_PERMANENT_REDIRECT:
        safe_status = 1;
        break;
    default:
        safe_status = 0;
        break;
    }

    // 处理 Expires 头部
    if (conf->expires != NGX_HTTP_EXPIRES_OFF && safe_status) {
        if (ngx_http_set_expires(r, conf) != NGX_OK) {
            return NGX_ERROR;
        }
    }

    // 处理自定义头部
    if (conf->headers) {
        h = conf->headers->elts;
        for (i = 0; i < conf->headers->nelts; i++) {

            if (!safe_status && !h[i].always) {
                continue;  // 非安全状态码且非 always 头部，跳过
            }

            // 计算头部值
            if (ngx_http_complex_value(r, &h[i].value, &value) != NGX_OK) {
                return NGX_ERROR;
            }

            // 添加或设置头部
            if (h[i].handler(r, &h[i], &value) != NGX_OK) {
                return NGX_ERROR;
            }
        }
    }

    return ngx_http_next_header_filter(r);
}
```

## 3. Chunked 过滤器

### 3.1 Chunked 过滤器实现

**源码位置**: `src/http/modules/ngx_http_chunked_filter_module.c` - `ngx_http_chunked_body_filter()`

Chunked 过滤器负责实现 HTTP/1.1 的分块传输编码。

**功能概述**：
Chunked 过滤器实现了 HTTP/1.1 的分块传输编码，支持流式响应传输：
- **分块编码**：将响应体分割成多个块，每个块包含大小信息
- **流式传输**：支持在不知道总长度的情况下传输响应
- **连接保持**：在不知道内容长度时仍能保持 HTTP/1.1 连接
- **实时响应**：支持服务器推送和实时数据流
- **协议兼容**：确保与 HTTP/1.1 协议的完全兼容

这个过滤器是 HTTP/1.1 长连接和流式传输的关键组件。

```c
static ngx_int_t ngx_http_chunked_body_filter(ngx_http_request_t *r, ngx_chain_t *in)
{
    u_char       *chunk;
    off_t         size;
    ngx_buf_t    *b;
    ngx_chain_t  *out, *cl, *tl, **ll;

    if (in == NULL || !r->chunked || r->header_only) {
        return ngx_http_next_body_filter(r, in);
    }

    out = NULL;
    ll = &out;

    // 处理每个输入缓冲区
    for (cl = in; cl; cl = cl->next) {

        // 计算块大小
        size = ngx_buf_size(cl->buf);

        if (size == 0 && !ngx_buf_special(cl->buf)) {
            continue;
        }

        // 创建块大小头部
        if (size > 0) {
            tl = ngx_alloc_chain_link(r->pool);
            if (tl == NULL) {
                return NGX_ERROR;
            }

            b = ngx_calloc_buf(r->pool);
            if (b == NULL) {
                return NGX_ERROR;
            }

            // 分配块头缓冲区
            chunk = ngx_palloc(r->pool, NGX_OFF_T_LEN + 2);
            if (chunk == NULL) {
                return NGX_ERROR;
            }

            // 格式化块大小（十六进制）
            b->pos = chunk;
            b->last = ngx_sprintf(chunk, "%xO" CRLF, size);
            b->memory = 1;

            tl->buf = b;
            *ll = tl;
            ll = &tl->next;
        }

        // 添加实际数据
        tl = ngx_alloc_chain_link(r->pool);
        if (tl == NULL) {
            return NGX_ERROR;
        }

        tl->buf = cl->buf;
        *ll = tl;
        ll = &tl->next;

        // 添加块结束标记
        if (size > 0) {
            tl = ngx_alloc_chain_link(r->pool);
            if (tl == NULL) {
                return NGX_ERROR;
            }

            b = ngx_calloc_buf(r->pool);
            if (b == NULL) {
                return NGX_ERROR;
            }

            b->pos = (u_char *) CRLF;
            b->last = b->pos + 2;
            b->memory = 1;

            tl->buf = b;
            *ll = tl;
            ll = &tl->next;
        }

        // 处理最后一块
        if (cl->buf->last_buf) {
            tl = ngx_alloc_chain_link(r->pool);
            if (tl == NULL) {
                return NGX_ERROR;
            }

            b = ngx_calloc_buf(r->pool);
            if (b == NULL) {
                return NGX_ERROR;
            }

            // 添加结束块 "0\r\n\r\n"
            b->pos = (u_char *) "0" CRLF CRLF;
            b->last = b->pos + 5;
            b->memory = 1;
            b->last_buf = 1;

            tl->buf = b;
            *ll = tl;
            ll = &tl->next;
        }
    }

    return ngx_http_next_body_filter(r, out);
}
```

## 4. 过滤器开发指南

### 4.1 自定义过滤器开发

开发自定义过滤器需要遵循 NGINX 的过滤器接口规范：

**基本结构**：
```c
// 过滤器函数指针
static ngx_http_output_header_filter_pt  ngx_http_next_header_filter;
static ngx_http_output_body_filter_pt    ngx_http_next_body_filter;

// 头部过滤器实现
static ngx_int_t ngx_http_custom_header_filter(ngx_http_request_t *r)
{
    // 处理响应头
    // ...

    // 调用下一个过滤器
    return ngx_http_next_header_filter(r);
}

// 内容过滤器实现
static ngx_int_t ngx_http_custom_body_filter(ngx_http_request_t *r, ngx_chain_t *in)
{
    // 处理响应体
    // ...

    // 调用下一个过滤器
    return ngx_http_next_body_filter(r, in);
}

// 过滤器注册
static ngx_int_t ngx_http_custom_filter_init(ngx_conf_t *cf)
{
    ngx_http_next_header_filter = ngx_http_top_header_filter;
    ngx_http_top_header_filter = ngx_http_custom_header_filter;

    ngx_http_next_body_filter = ngx_http_top_body_filter;
    ngx_http_top_body_filter = ngx_http_custom_body_filter;

    return NGX_OK;
}
```

### 4.2 过滤器最佳实践

1. **性能考虑**：
   - 尽早判断是否需要处理，避免不必要的计算
   - 使用内存池分配临时内存，避免内存泄漏
   - 支持流式处理，避免缓存大量数据

2. **错误处理**：
   - 正确处理内存分配失败
   - 在出错时调用下一个过滤器或返回错误
   - 记录适当的错误日志

3. **兼容性**：
   - 检查请求和响应的状态
   - 处理子请求的情况
   - 考虑与其他过滤器的交互

## 5. 总结

NGINX 的响应过滤器实现展现了其强大的模块化设计和高性能特性：

1. **Gzip 过滤器**：提供了高效的内容压缩，显著减少网络传输量
2. **Headers 过滤器**：提供了灵活的响应头管理，支持各种定制需求
3. **Chunked 过滤器**：实现了 HTTP/1.1 分块传输，支持流式响应
4. **过滤器链机制**：通过责任链模式实现了模块化和可扩展性

理解这些过滤器的实现原理对于：
- **性能优化**：合理配置过滤器参数
- **功能扩展**：开发自定义过滤器
- **问题诊断**：分析响应处理问题
- **架构设计**：借鉴过滤器链的设计思想

都具有重要的指导意义。NGINX 的过滤器机制为 Web 服务器的响应处理提供了一个优秀的架构范例。