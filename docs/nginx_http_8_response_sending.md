# NGINX 响应发送阶段深度解析

## 概述

本文是 NGINX HTTP 请求处理流程系列的第五篇，重点分析响应发送阶段的实现机制。响应发送阶段是整个请求处理流程的关键环节，负责将经过过滤器链处理的响应数据高效地传输给客户端。这个阶段体现了 NGINX 在网络 I/O 处理方面的卓越性能优化。

## 1. 网络 I/O 处理机制

### 1.1 发送链函数

**源码位置**: `src/os/unix/ngx_linux_sendfile_chain.c` - `ngx_linux_sendfile_chain()`

发送链函数是 NGINX 网络传输的核心，负责将数据链高效地发送给客户端。

**功能概述**：
发送链函数是 NGINX 高性能网络传输的核心组件，实现了多种优化策略：
- **零拷贝传输**：利用 sendfile 系统调用实现文件到套接字的零拷贝传输
- **向量化 I/O**：使用 writev 系统调用批量发送多个缓冲区
- **智能缓冲**：根据数据类型选择最优的传输方式
- **流量控制**：支持传输限制和带宽控制
- **错误恢复**：处理网络中断和重传机制

这种设计使得 NGINX 能够在高并发场景下保持卓越的网络传输性能。

```c
ngx_chain_t *ngx_linux_sendfile_chain(ngx_connection_t *c, ngx_chain_t *in, off_t limit)
{
    int            tcp_nodelay;
    off_t          send, prev_send;
    size_t         file_size, sent;
    ssize_t        n;
    ngx_err_t      err;
    ngx_buf_t     *file;
    ngx_event_t   *wev;
    ngx_chain_t   *cl;
    ngx_iovec_t    header;
    struct iovec   headers[NGX_IOVS_PREALLOCATE];

    wev = c->write;

    if (!wev->ready) {
        return in;  // 套接字不可写，返回未发送的链
    }

    // 设置传输限制
    if (limit == 0 || limit > (off_t) (NGX_SENDFILE_MAXSIZE - ngx_pagesize)) {
        limit = NGX_SENDFILE_MAXSIZE - ngx_pagesize;
    }

    send = 0;
    header.iovs = headers;
    header.nalloc = NGX_IOVS_PREALLOCATE;

    // 遍历数据链，分类处理内存数据和文件数据
    for (cl = in; cl && send < limit; cl = cl->next) {

        if (ngx_buf_special(cl->buf)) {
            continue;  // 跳过特殊缓冲区
        }

        if (!ngx_buf_in_memory(cl->buf) && !cl->buf->in_file) {
            continue;  // 跳过空缓冲区
        }

        if (ngx_buf_in_memory(cl->buf)) {
            // 内存数据：添加到向量 I/O 数组
            if (header.count == header.nalloc) {
                break;  // 向量数组已满
            }

            header.iovs[header.count].iov_base = (void *) cl->buf->pos;
            header.iovs[header.count].iov_len = cl->buf->last - cl->buf->pos;
            header.count++;
            send += cl->buf->last - cl->buf->pos;

        } else {
            // 文件数据：使用 sendfile 零拷贝传输
            file = cl->buf;
            file_size = (size_t) (file->file_last - file->file_pos);

            if (send + file_size > limit) {
                file_size = (size_t) (limit - send);
            }

            // 先发送内存数据
            if (header.count) {
                n = ngx_writev(c, &header);
                if (n == NGX_ERROR) {
                    return NGX_CHAIN_ERROR;
                }
                sent = (n > 0) ? n : 0;
                c->sent += sent;
            }

            // 发送文件数据
            n = ngx_linux_sendfile(c, file, file_size);
            if (n == NGX_ERROR) {
                return NGX_CHAIN_ERROR;
            }

            sent = (n > 0) ? n : 0;
            c->sent += sent;
            send += sent;

            if (n != (ssize_t) file_size) {
                wev->ready = 0;  // 套接字缓冲区已满
                return cl;
            }
        }
    }

    // 发送剩余的内存数据
    if (header.count) {
        n = ngx_writev(c, &header);
        if (n == NGX_ERROR) {
            return NGX_CHAIN_ERROR;
        }
        sent = (n > 0) ? n : 0;
        c->sent += sent;
    }

    return cl;  // 返回未发送完的链
}
```

### 1.2 基础发送函数

**源码位置**: `src/os/unix/ngx_send.c` - `ngx_unix_send()`

基础发送函数提供了最底层的数据发送功能。

**功能概述**：
基础发送函数是网络传输的基础层，提供可靠的数据发送服务：
- **系统调用封装**：封装 send() 系统调用，提供统一的接口
- **错误处理**：处理各种网络错误和异常情况
- **非阻塞支持**：支持非阻塞 I/O 模式
- **重试机制**：处理信号中断和临时错误
- **统计信息**：记录发送字节数和连接状态

这个函数确保了网络传输的可靠性和稳定性。

```c
ssize_t ngx_unix_send(ngx_connection_t *c, u_char *buf, size_t size)
{
    ssize_t       n;
    ngx_err_t     err;
    ngx_event_t  *wev;

    wev = c->write;

    for ( ;; ) {
        // 调用系统 send() 函数
        n = send(c->fd, buf, size, 0);

        if (n > 0) {
            // 发送成功
            if (n < (ssize_t) size) {
                wev->ready = 0;  // 套接字缓冲区已满
            }
            c->sent += n;  // 更新发送统计
            return n;
        }

        err = ngx_socket_errno;

        if (n == 0) {
            // 连接被对端关闭
            ngx_log_error(NGX_LOG_ALERT, c->log, err, "send() returned zero");
            wev->ready = 0;
            return n;
        }

        if (err == NGX_EAGAIN || err == NGX_EINTR) {
            wev->ready = 0;

            if (err == NGX_EAGAIN) {
                return NGX_AGAIN;  // 需要等待套接字可写
            }

            // 信号中断，继续重试
        } else {
            // 发送错误
            wev->error = 1;
            (void) ngx_connection_error(c, err, "send() failed");
            return NGX_ERROR;
        }
    }
}
```

## 2. 发送缓冲区管理

### 2.1 输出链管理

**源码位置**: `src/core/ngx_output_chain.c` - `ngx_output_chain()`

输出链管理负责高效地管理待发送的数据缓冲区。

**功能概述**：
输出链管理是 NGINX 缓冲区管理的核心，实现了高效的内存使用策略：
- **缓冲区复用**：通过缓冲区池减少内存分配和释放开销
- **零拷贝优化**：尽可能避免数据拷贝，直接传递缓冲区指针
- **流式处理**：支持大文件的流式传输，避免内存占用过大
- **智能缓冲**：根据数据特征选择合适的缓冲策略
- **内存控制**：通过限制缓冲区大小控制内存使用

这种设计确保了 NGINX 在处理大量并发连接时的内存效率。

```c
ngx_int_t ngx_output_chain(ngx_output_chain_ctx_t *ctx, ngx_chain_t *in)
{
    off_t         bsize;
    ngx_int_t     rc, last;
    ngx_chain_t  *cl, *out, **last_out;

    // 快速路径：直接传递数据，无需拷贝
    if (ctx->in == NULL && ctx->busy == NULL && !ctx->aio) {
        if (in == NULL) {
            return ctx->output_filter(ctx->filter_ctx, in);
        }

        if (in->next == NULL && ngx_output_chain_as_is(ctx, in->buf)) {
            return ctx->output_filter(ctx->filter_ctx, in);
        }
    }

    last = NGX_NONE;
    out = NULL;
    last_out = &out;

    // 处理输入链
    for ( ;; ) {
        while (ctx->in) {
            // 获取缓冲区大小
            bsize = ngx_buf_size(ctx->in->buf);

            if (bsize == 0 && !ngx_buf_special(ctx->in->buf)) {
                // 跳过空缓冲区
                ctx->in = ctx->in->next;
                continue;
            }

            // 检查是否可以直接使用缓冲区
            if (ngx_output_chain_as_is(ctx, ctx->in->buf)) {
                // 直接添加到输出链
                cl = ngx_alloc_chain_link(ctx->pool);
                if (cl == NULL) {
                    return NGX_ERROR;
                }

                cl->buf = ctx->in->buf;
                cl->next = NULL;
                *last_out = cl;
                last_out = &cl->next;
                ctx->in = ctx->in->next;

            } else {
                // 需要拷贝数据到临时缓冲区
                if (ctx->buf == NULL) {
                    rc = ngx_output_chain_align_file_buf(ctx, bsize);
                    if (rc == NGX_ERROR) {
                        return NGX_ERROR;
                    }
                }

                rc = ngx_output_chain_copy_buf(ctx);
                if (rc == NGX_ERROR) {
                    return NGX_ERROR;
                }

                if (rc == NGX_AGAIN) {
                    if (out) {
                        break;
                    }
                    return rc;
                }

                // 添加拷贝后的缓冲区到输出链
                cl = ngx_alloc_chain_link(ctx->pool);
                if (cl == NULL) {
                    return NGX_ERROR;
                }

                cl->buf = ctx->buf;
                cl->next = NULL;
                *last_out = cl;
                last_out = &cl->next;
                ctx->buf = NULL;
            }
        }

        if (out == NULL && last != NGX_NONE) {
            if (ctx->in) {
                return NGX_AGAIN;
            }
            return last;
        }

        // 调用输出过滤器发送数据
        last = ctx->output_filter(ctx->filter_ctx, out);

        if (last == NGX_ERROR || last == NGX_DONE) {
            return last;
        }

        // 更新缓冲区链状态
        ngx_chain_update_chains(ctx->pool, &ctx->free, &ctx->busy, &out, ctx->tag);
        last_out = &out;
    }
}
```

## 3. Sendfile 零拷贝优化

### 3.1 Sendfile 实现

**源码位置**: `src/os/unix/ngx_linux_sendfile_chain.c` - `ngx_linux_sendfile()`

Sendfile 是 NGINX 实现零拷贝传输的关键技术。

**功能概述**：
Sendfile 零拷贝技术是 NGINX 高性能文件传输的核心，显著提升了静态文件服务的效率：
- **零拷贝传输**：数据直接从文件系统缓存传输到网络套接字，避免用户空间拷贝
- **CPU 效率**：减少 CPU 使用率，释放更多计算资源处理其他请求
- **内存节省**：避免在用户空间分配大量缓冲区
- **异步支持**：支持异步文件 I/O，提高并发处理能力
- **错误处理**：完善的错误处理和重试机制

这项技术使得 NGINX 在静态文件服务方面具有卓越的性能优势。

```c
static ssize_t ngx_linux_sendfile(ngx_connection_t *c, ngx_buf_t *file, size_t size)
{
    ssize_t    n;
    ngx_err_t  err;
    off_t      offset;

#if (NGX_HAVE_SENDFILE64)
    offset = file->file_pos;
#else
    offset = (int32_t) file->file_pos;
#endif

eintr:
    // 调用 sendfile 系统调用
    n = sendfile(c->fd, file->file->fd, &offset, size);

    if (n == -1) {
        err = ngx_errno;

        switch (err) {
        case NGX_EAGAIN:
            // 套接字缓冲区已满，需要等待
            return NGX_AGAIN;

        case NGX_EINTR:
            // 信号中断，重试
            goto eintr;

        default:
            // 发送错误
            c->write->error = 1;
            ngx_connection_error(c, err, "sendfile() failed");
            return NGX_ERROR;
        }
    }

    // 更新文件位置和发送统计
    if (n >= 0) {
        file->file_pos += n;
        c->sent += n;
    }

    return n;
}
```

### 3.2 向量化 I/O

**源码位置**: `src/os/unix/ngx_writev_chain.c` - `ngx_writev_chain()`

向量化 I/O 允许一次系统调用发送多个缓冲区。

**功能概述**：
向量化 I/O 是 NGINX 批量数据传输的重要优化技术：
- **批量传输**：一次系统调用发送多个不连续的内存缓冲区
- **系统调用优化**：减少系统调用次数，降低内核态切换开销
- **内存效率**：避免将多个缓冲区合并到单一缓冲区
- **灵活组合**：支持内存数据和文件数据的混合传输
- **性能提升**：在小文件和动态内容传输中显著提升性能

这种技术特别适合传输由多个小块组成的响应数据。

```c
ngx_chain_t *ngx_writev_chain(ngx_connection_t *c, ngx_chain_t *in, off_t limit)
{
    ssize_t        n, sent;
    off_t          send, prev_send;
    ngx_chain_t   *cl;
    ngx_event_t   *wev;
    ngx_iovec_t    vec;
    struct iovec   iovs[NGX_IOVS_PREALLOCATE];

    wev = c->write;

    if (!wev->ready) {
        return in;
    }

    // 设置传输限制
    if (limit == 0 || limit > NGX_SENDFILE_MAXSIZE) {
        limit = NGX_SENDFILE_MAXSIZE;
    }

    send = 0;

    // 构建向量 I/O 数组
    vec.iovs = iovs;
    vec.nalloc = NGX_IOVS_PREALLOCATE;

    for (cl = in; cl && send < limit; cl = cl->next) {

        if (ngx_buf_special(cl->buf)) {
            continue;
        }

        if (!ngx_buf_in_memory(cl->buf)) {
            break;  // 遇到文件缓冲区，停止
        }

        if (vec.count == vec.nalloc) {
            break;  // 向量数组已满
        }

        size = cl->buf->last - cl->buf->pos;

        if (send + size > limit) {
            size = (size_t) (limit - send);
        }

        // 添加到向量数组
        iovs[vec.count].iov_base = (void *) cl->buf->pos;
        iovs[vec.count].iov_len = size;
        vec.count++;
        send += size;
    }

    // 执行向量化写入
    n = ngx_writev(c, &vec);

    if (n == NGX_ERROR) {
        return NGX_CHAIN_ERROR;
    }

    sent = (n > 0) ? n : 0;
    c->sent += sent;

    // 更新缓冲区位置
    for (cl = in; cl; cl = cl->next) {

        if (ngx_buf_special(cl->buf)) {
            continue;
        }

        if (!ngx_buf_in_memory(cl->buf)) {
            break;
        }

        size = cl->buf->last - cl->buf->pos;

        if (sent >= size) {
            sent -= size;
            cl->buf->pos = cl->buf->last;
        } else {
            cl->buf->pos += sent;
            break;
        }
    }

    return cl;
}
```

## 4. 连接状态管理

### 4.1 写事件处理

**源码位置**: `src/http/ngx_http_request.c` - `ngx_http_writer()`

写事件处理器负责管理响应发送过程中的连接状态。

**功能概述**：
写事件处理器是响应发送的状态管理中心，确保数据传输的可靠性：
- **事件驱动**：基于 epoll 等事件机制，高效处理可写事件
- **流量控制**：根据网络状况动态调整发送速度
- **超时管理**：处理发送超时，防止连接长时间占用资源
- **错误恢复**：处理网络错误和连接中断
- **状态同步**：维护连接和请求的状态一致性

这个处理器确保了响应发送过程的稳定性和可靠性。

```c
static void ngx_http_writer(ngx_event_t *wev)
{
    ngx_int_t                  rc;
    ngx_connection_t          *c;
    ngx_http_request_t        *r;
    ngx_http_core_loc_conf_t  *clcf;

    c = wev->data;
    r = c->data;

    // 检查连接状态
    if (c->error || c->timedout || c->close || c->destroyed) {
        ngx_http_finalize_request(r, NGX_HTTP_CLIENT_CLOSED_REQUEST);
        return;
    }

    // 检查写事件超时
    if (wev->timedout) {
        ngx_log_error(NGX_LOG_INFO, c->log, NGX_ETIMEDOUT, "client timed out");
        c->timedout = 1;
        ngx_http_finalize_request(r, NGX_HTTP_REQUEST_TIME_OUT);
        return;
    }

    // 发送待发送的数据
    rc = ngx_http_output_filter(r, NULL);

    if (rc == NGX_ERROR) {
        ngx_http_finalize_request(r, rc);
        return;
    }

    if (r->buffered || r->postponed || (r == r->main && c->buffered)) {
        // 还有数据待发送
        if (!wev->delayed) {
            clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);
            ngx_add_timer(wev, clcf->send_timeout);  // 设置发送超时
        }

        if (ngx_handle_write_event(wev, clcf->send_lowat) != NGX_OK) {
            ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
            return;
        }

        return;
    }

    // 数据发送完成
    ngx_log_debug2(NGX_LOG_DEBUG_HTTP, wev->log, 0,
                   "http writer done: \"%V?%V\"", &r->uri, &r->args);

    r->write_event_handler = ngx_http_request_empty_handler;

    ngx_http_finalize_request(r, rc);
}
```

## 5. 错误处理和重试机制

### 5.1 网络错误处理

NGINX 在响应发送过程中实现了完善的错误处理机制：

**错误类型处理**：
- **EAGAIN**：套接字缓冲区已满，等待下次可写事件
- **EINTR**：信号中断，立即重试发送操作
- **EPIPE**：客户端关闭连接，终止发送并清理资源
- **ECONNRESET**：连接被重置，记录日志并关闭连接

**重试策略**：
- **自动重试**：对于临时性错误（如 EINTR）自动重试
- **延迟重试**：对于资源不足错误，等待资源可用后重试
- **限制重试**：设置重试次数限制，避免无限重试

### 5.2 超时管理

NGINX 通过多层超时机制确保连接不会长时间占用资源：

**超时类型**：
- **发送超时**：`send_timeout` 控制单次发送操作的最大时间
- **连接超时**：`keepalive_timeout` 控制连接保持的最大时间
- **客户端超时**：检测客户端是否仍然活跃

## 6. 总结

NGINX 的响应发送阶段展现了其在网络 I/O 处理方面的卓越设计：

1. **零拷贝优化**：通过 sendfile 技术实现高效的文件传输
2. **向量化 I/O**：通过 writev 减少系统调用开销
3. **智能缓冲**：通过输出链管理实现高效的内存使用
4. **事件驱动**：基于 epoll 的异步非阻塞 I/O 模型
5. **错误恢复**：完善的错误处理和重试机制

这些技术的综合运用使得 NGINX 能够：
- **高并发处理**：同时处理数万个并发连接
- **低资源消耗**：高效利用 CPU 和内存资源
- **高可靠性**：稳定处理各种网络异常情况
- **优秀性能**：在静态文件服务和反向代理场景下表现卓越

理解响应发送阶段的实现原理对于 NGINX 的性能调优、问题诊断和架构设计都具有重要意义。
