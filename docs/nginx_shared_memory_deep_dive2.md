# Nginx Master-Worker 进程间共享内存实现深度解析

## 概述

在nginx的多进程架构中，master进程负责管理worker进程，而worker进程负责处理实际的客户端请求。为了实现进程间的数据共享和协调，nginx使用了共享内存机制。本文将深入分析nginx共享内存的实现原理、数据结构、同步机制以及具体应用场景。

## 1. 共享内存基础架构

### 1.1 核心数据结构

nginx共享内存的核心数据结构是`ngx_shm_t`，它定义了共享内存段的基本属性：

```c
// 共享内存基础结构体 - 定义共享内存段的基本属性
typedef struct {
    u_char      *addr;      // 共享内存起始地址
    size_t       size;      // 共享内存大小
    ngx_str_t    name;      // 共享内存名称
    ngx_log_t   *log;       // 日志对象
    ngx_uint_t   exists;    // 是否已存在标志
#if (NGX_WIN32)
    HANDLE       handle;    // Windows平台的句柄
#endif
} ngx_shm_t;
```

### 1.2 平台特定实现

nginx针对不同操作系统提供了不同的共享内存实现方式：

#### Unix/Linux平台 - mmap实现
在支持MAP_ANON的Unix系统上，nginx使用mmap系统调用创建匿名共享内存：

```c
// Unix平台使用mmap创建匿名共享内存
ngx_int_t ngx_shm_alloc(ngx_shm_t *shm)
{
    // 使用mmap创建匿名共享内存映射
    shm->addr = (u_char *) mmap(NULL, shm->size,
                                PROT_READ|PROT_WRITE,    // 读写权限
                                MAP_ANON|MAP_SHARED,     // 匿名共享映射
                                -1, 0);
    
    if (shm->addr == MAP_FAILED) {
        ngx_log_error(NGX_LOG_ALERT, shm->log, ngx_errno,
                      "mmap(MAP_ANON|MAP_SHARED, %uz) failed", shm->size);
        return NGX_ERROR;
    }
    
    return NGX_OK;
}
```

#### System V共享内存实现
在不支持MAP_ANON的系统上，nginx使用System V IPC机制：

```c
// System V IPC实现共享内存
ngx_int_t ngx_shm_alloc(ngx_shm_t *shm)
{
    int id;
    
    // 创建System V共享内存段
    id = shmget(IPC_PRIVATE, shm->size, (SHM_R|SHM_W|IPC_CREAT));
    if (id == -1) {
        return NGX_ERROR;
    }
    
    // 将共享内存段附加到进程地址空间
    shm->addr = shmat(id, NULL, 0);
    
    // 标记共享内存段为删除（进程退出时自动清理）
    if (shmctl(id, IPC_RMID, NULL) == -1) {
        ngx_log_error(NGX_LOG_ALERT, shm->log, ngx_errno,
                      "shmctl(IPC_RMID) failed");
    }
    
    return (shm->addr == (void *) -1) ? NGX_ERROR : NGX_OK;
}
```

#### Windows平台实现
Windows平台使用CreateFileMapping API创建共享内存：

```c
// Windows平台使用CreateFileMapping创建共享内存
ngx_int_t ngx_shm_alloc(ngx_shm_t *shm)
{
    // 创建文件映射对象
    shm->handle = CreateFileMapping(INVALID_HANDLE_VALUE, NULL, 
                                    PAGE_READWRITE,
                                    (u_long) (shm->size >> 32),
                                    (u_long) (shm->size & 0xffffffff),
                                    (char *) name);
    
    if (shm->handle == NULL) {
        return NGX_ERROR;
    }
    
    // 将文件映射对象映射到进程地址空间
    shm->addr = MapViewOfFile(shm->handle, FILE_MAP_WRITE, 0, 0, 0);
    
    return (shm->addr == NULL) ? NGX_ERROR : NGX_OK;
}
```

## 2. 共享内存区域管理

### 2.1 共享内存区域结构

nginx使用`ngx_shm_zone_t`结构体来管理具体的共享内存区域：

```c
// 共享内存区域管理结构体
struct ngx_shm_zone_s {
    void                     *data;      // 区域特定数据
    ngx_shm_t                 shm;       // 底层共享内存
    ngx_shm_zone_init_pt      init;      // 初始化回调函数
    void                     *tag;       // 标识标签
    void                     *sync;      // 同步对象
    ngx_uint_t                noreuse;   // 不可重用标志
};
```

### 2.2 共享内存区域创建

共享内存区域通过`ngx_shared_memory_add`函数创建和管理：

```c
// 创建或获取共享内存区域
ngx_shm_zone_t *ngx_shared_memory_add(ngx_conf_t *cf, ngx_str_t *name, 
                                       size_t size, void *tag)
{
    ngx_shm_zone_t   *shm_zone;
    ngx_list_part_t  *part;
    
    // 从配置周期中获取共享内存列表
    part = &cf->cycle->shared_memory.part;
    shm_zone = part->elts;
    
    // 检查是否已存在同名的共享内存区域
    for (i = 0; /* void */; i++) {
        if (ngx_strcmp(name->data, shm_zone[i].shm.name.data) == 0) {
            // 检查大小是否匹配
            if (size && size != shm_zone[i].shm.size) {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                "the size %uz of shared memory zone \"%V\" "
                                "conflicts with already declared size %uz",
                                size, &shm_zone[i].shm.name, 
                                shm_zone[i].shm.size);
                return NULL;
            }
            return &shm_zone[i];
        }
    }
    
    // 创建新的共享内存区域
    shm_zone = ngx_list_push(&cf->cycle->shared_memory);
    if (shm_zone == NULL) {
        return NULL;
    }
    
    // 初始化共享内存区域属性
    shm_zone->data = NULL;
    shm_zone->shm.log = cf->cycle->log;
    shm_zone->shm.addr = NULL;
    shm_zone->shm.size = size;
    shm_zone->shm.name = *name;
    shm_zone->shm.exists = 0;
    shm_zone->init = NULL;
    shm_zone->tag = tag;
    shm_zone->noreuse = 0;
    
    return shm_zone;
}
```

## 3. Slab分配器实现

### 3.1 Slab分配器结构

nginx在共享内存之上实现了slab分配器，用于高效的内存分配和管理：

```c
// Slab内存池结构体
typedef struct {
    ngx_shmtx_sh_t    lock;         // 共享内存锁
    
    size_t            min_size;     // 最小分配单位
    size_t            min_shift;    // 最小分配单位的位移值
    
    ngx_slab_page_t  *pages;        // 页面数组
    ngx_slab_page_t  *last;         // 最后一个页面
    ngx_slab_page_t   free;         // 空闲页面链表
    
    ngx_slab_stat_t  *stats;        // 统计信息
    ngx_uint_t        pfree;        // 空闲页面数量
    
    u_char           *start;        // 内存池起始地址
    u_char           *end;          // 内存池结束地址
    
    ngx_shmtx_t       mutex;        // 互斥锁
    
    u_char           *log_ctx;      // 日志上下文
    u_char            zero;         // 零字节
    
    unsigned          log_nomem:1;  // 内存不足日志标志
    
    void             *data;         // 用户数据
    void             *addr;         // 内存池地址
} ngx_slab_pool_t;
```

### 3.2 Slab内存分配

Slab分配器提供了线程安全的内存分配接口：

```c
// Slab内存分配 - 带锁保护的分配函数
void *ngx_slab_alloc(ngx_slab_pool_t *pool, size_t size)
{
    void  *p;

    // 获取共享内存互斥锁
    ngx_shmtx_lock(&pool->mutex);

    // 执行实际的内存分配
    p = ngx_slab_alloc_locked(pool, size);

    // 释放共享内存互斥锁
    ngx_shmtx_unlock(&pool->mutex);

    return p;
}

// Slab内存释放 - 带锁保护的释放函数
void ngx_slab_free(ngx_slab_pool_t *pool, void *p)
{
    // 获取共享内存互斥锁
    ngx_shmtx_lock(&pool->mutex);

    // 执行实际的内存释放
    ngx_slab_free_locked(pool, p);

    // 释放共享内存互斥锁
    ngx_shmtx_unlock(&pool->mutex);
}
```

### 3.3 Slab初始化

Slab分配器在共享内存区域初始化时进行设置：

```c
// 初始化共享内存区域的Slab分配器
static ngx_int_t ngx_init_zone_pool(ngx_cycle_t *cycle, ngx_shm_zone_t *zn)
{
    ngx_slab_pool_t  *sp;

    // 将共享内存起始地址转换为slab池
    sp = (ngx_slab_pool_t *) zn->shm.addr;

    if (zn->shm.exists) {
        // 共享内存已存在，检查地址一致性
        if (sp == sp->addr) {
            return NGX_OK;
        }

        ngx_log_error(NGX_LOG_EMERG, cycle->log, 0,
                      "shared zone \"%V\" has no equal addresses: %p vs %p",
                      &zn->shm.name, sp->addr, sp);
        return NGX_ERROR;
    }

    // 初始化slab池属性
    sp->end = zn->shm.addr + zn->shm.size;  // 设置结束地址
    sp->min_shift = 3;                       // 最小分配单位为8字节
    sp->addr = zn->shm.addr;                // 设置起始地址

    // 创建共享内存互斥锁
    if (ngx_shmtx_create(&sp->mutex, &sp->lock, file) != NGX_OK) {
        return NGX_ERROR;
    }

    // 初始化slab分配器
    ngx_slab_init(sp);

    return NGX_OK;
}
```

## 4. 共享内存同步机制

### 4.1 共享内存互斥锁

nginx使用`ngx_shmtx_t`结构体实现共享内存互斥锁：

```c
// 共享内存互斥锁结构体
typedef struct {
#if (NGX_HAVE_ATOMIC_OPS)
    ngx_atomic_t  *lock;        // 原子锁变量
#if (NGX_HAVE_POSIX_SEM)
    ngx_atomic_t  *wait;        // 等待计数器
    ngx_uint_t     semaphore;   // 信号量标志
    sem_t          sem;         // POSIX信号量
#endif
#else
    ngx_fd_t       fd;          // 文件描述符（用于文件锁）
    u_char        *name;        // 锁文件名
#endif
    ngx_uint_t     spin;        // 自旋次数
} ngx_shmtx_t;
```

### 4.2 互斥锁操作

共享内存互斥锁提供了基本的锁操作：

```c
// 尝试获取锁 - 非阻塞方式
ngx_uint_t ngx_shmtx_trylock(ngx_shmtx_t *mtx)
{
    // 使用原子比较交换操作尝试获取锁
    return (*mtx->lock == 0 && ngx_atomic_cmp_set(mtx->lock, 0, ngx_pid));
}

// 获取锁 - 阻塞方式，包含自旋等待
void ngx_shmtx_lock(ngx_shmtx_t *mtx)
{
    ngx_uint_t i, n;

    for ( ;; ) {
        // 尝试获取锁
        if (*mtx->lock == 0 && ngx_atomic_cmp_set(mtx->lock, 0, ngx_pid)) {
            return;
        }

        // 在多CPU系统上进行自旋等待
        if (ngx_ncpu > 1) {
            for (n = 1; n < mtx->spin; n <<= 1) {
                // 指数退避自旋
                for (i = 0; i < n; i++) {
                    ngx_cpu_pause();  // CPU暂停指令
                }

                // 再次尝试获取锁
                if (*mtx->lock == 0
                    && ngx_atomic_cmp_set(mtx->lock, 0, ngx_pid))
                {
                    return;
                }
            }
        }

        // 自旋失败后使用信号量等待
        ngx_shmtx_wakeup(mtx);
    }
}

// 释放锁
void ngx_shmtx_unlock(ngx_shmtx_t *mtx)
{
    if (ngx_atomic_cmp_set(mtx->lock, ngx_pid, 0)) {
        ngx_shmtx_wakeup(mtx);  // 唤醒等待的进程
    }
}
```

## 5. 核心应用场景

### 5.1 Accept Mutex（接受互斥锁）

Accept mutex是nginx中最重要的共享内存应用之一，用于协调多个worker进程对监听套接字的访问：

```c
// Accept mutex的初始化 - 在事件模块初始化时创建
static ngx_int_t ngx_event_module_init(ngx_cycle_t *cycle)
{
    void            ***cf;
    u_char           *shared;
    size_t            size, cl;
    ngx_shm_t         shm;
    ngx_time_t       *tp;
    ngx_core_conf_t  *ccf;
    ngx_event_conf_t *ecf;

    // 计算共享内存大小（缓存行对齐）
    cl = 128;  // 缓存行大小
    size = cl            /* accept mutex */
           + cl          /* connection counter */
           + cl;         /* temp number */

    // 创建nginx核心共享内存区域
    shm.size = size;
    ngx_str_set(&shm.name, "nginx_shared_zone");
    shm.log = cycle->log;

    if (ngx_shm_alloc(&shm) != NGX_OK) {
        return NGX_ERROR;
    }

    shared = shm.addr;

    // 初始化accept mutex
    ngx_accept_mutex_ptr = (ngx_atomic_t *) shared;
    ngx_accept_mutex.spin = (ngx_uint_t) -1;

    if (ngx_shmtx_create(&ngx_accept_mutex, (ngx_shmtx_sh_t *) shared,
                         cycle->lock_file.data) != NGX_OK)
    {
        return NGX_ERROR;
    }

    // 初始化连接计数器
    ngx_connection_counter = (ngx_atomic_t *) (shared + 1 * cl);
    (void) ngx_atomic_cmp_set(ngx_connection_counter, 0, 1);

    // 初始化临时文件编号
    ngx_temp_number = (ngx_atomic_t *) (shared + 2 * cl);

    return NGX_OK;
}
```

### 5.2 连接计数器

连接计数器用于跟踪全局连接数，帮助实现连接限制和负载均衡：

```c
// 在worker进程初始化时设置accept mutex使用
static ngx_int_t ngx_event_process_init(ngx_cycle_t *cycle)
{
    ngx_core_conf_t  *ccf;
    ngx_event_conf_t *ecf;

    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);
    ecf = ngx_event_get_conf(cycle->conf_ctx, ngx_event_core_module);

    // 判断是否需要使用accept mutex
    if (ccf->master && ccf->worker_processes > 1 && ecf->accept_mutex) {
        ngx_use_accept_mutex = 1;           // 启用accept mutex
        ngx_accept_mutex_held = 0;          // 当前未持有锁
        ngx_accept_mutex_delay = ecf->accept_mutex_delay;  // 设置延迟时间
    } else {
        ngx_use_accept_mutex = 0;           // 禁用accept mutex
    }

    return NGX_OK;
}
```

### 5.3 限流模块中的共享内存应用

#### Limit Request模块

limit_req模块使用共享内存实现跨worker进程的请求频率限制：

```c
// limit_req模块的共享内存初始化
static ngx_int_t ngx_http_limit_req_init_zone(ngx_shm_zone_t *shm_zone, void *data)
{
    ngx_http_limit_req_ctx_t  *octx = data;
    ngx_http_limit_req_ctx_t  *ctx;

    // 获取slab分配器
    ctx->shpool = (ngx_slab_pool_t *) shm_zone->shm.addr;

    if (shm_zone->shm.exists) {
        // 共享内存已存在，获取已有数据
        ctx->sh = ctx->shpool->data;
        return NGX_OK;
    }

    // 分配共享内存中的控制结构
    ctx->sh = ngx_slab_alloc(ctx->shpool, sizeof(ngx_http_limit_req_shctx_t));
    if (ctx->sh == NULL) {
        return NGX_ERROR;
    }

    ctx->shpool->data = ctx->sh;

    // 初始化红黑树用于存储请求记录
    ngx_rbtree_init(&ctx->sh->rbtree, &ctx->sh->sentinel,
                    ngx_http_limit_req_rbtree_insert_value);

    // 初始化LRU队列
    ngx_queue_init(&ctx->sh->queue);

    return NGX_OK;
}
```

#### Limit Connection模块

limit_conn模块类似地使用共享内存跟踪连接数：

```c
// limit_conn模块的共享内存初始化
static ngx_int_t ngx_http_limit_conn_init_zone(ngx_shm_zone_t *shm_zone, void *data)
{
    ngx_http_limit_conn_ctx_t  *ctx;

    ctx->shpool = (ngx_slab_pool_t *) shm_zone->shm.addr;

    if (shm_zone->shm.exists) {
        ctx->sh = ctx->shpool->data;
        return NGX_OK;
    }

    // 分配共享控制结构
    ctx->sh = ngx_slab_alloc(ctx->shpool, sizeof(ngx_http_limit_conn_shctx_t));
    if (ctx->sh == NULL) {
        return NGX_ERROR;
    }

    ctx->shpool->data = ctx->sh;

    // 初始化红黑树存储连接记录
    ngx_rbtree_init(&ctx->sh->rbtree, &ctx->sh->sentinel,
                    ngx_http_limit_conn_rbtree_insert_value);

    return NGX_OK;
}
```

### 5.4 Upstream Zone模块

upstream zone模块使用共享内存在worker进程间共享upstream服务器状态：

```c
// upstream zone的共享内存配置
static char *ngx_http_upstream_zone(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_str_t                    *value;
    ngx_http_upstream_srv_conf_t *uscf;
    ngx_http_upstream_main_conf_t *umcf;
    ssize_t                       size;

    value = cf->args->elts;

    // 解析zone大小参数
    if (cf->args->nelts == 3) {
        size = ngx_parse_size(&value[2]);

        if (size == NGX_ERROR) {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "invalid zone size \"%V\"", &value[2]);
            return NGX_CONF_ERROR;
        }

        // 检查最小大小限制
        if (size < (ssize_t) (8 * ngx_pagesize)) {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "zone \"%V\" is too small", &value[1]);
            return NGX_CONF_ERROR;
        }
    } else {
        size = 0;
    }

    // 创建共享内存区域
    uscf->shm_zone = ngx_shared_memory_add(cf, &value[1], size,
                                           &ngx_http_upstream_module);
    if (uscf->shm_zone == NULL) {
        return NGX_CONF_ERROR;
    }

    // 设置初始化回调和数据
    uscf->shm_zone->init = ngx_http_upstream_init_zone;
    uscf->shm_zone->data = umcf;
    uscf->shm_zone->noreuse = 1;  // 不允许重用

    return NGX_CONF_OK;
}
```

## 6. 初始化流程详解

### 6.1 Master进程中的共享内存创建

Master进程在启动时负责创建所有的共享内存区域：

```c
// 在nginx主循环中初始化共享内存
ngx_cycle_t *ngx_init_cycle(ngx_cycle_t *old_cycle)
{
    ngx_uint_t           i;
    ngx_list_part_t     *part;
    ngx_shm_zone_t      *shm_zone;

    // 获取共享内存区域列表
    part = &cycle->shared_memory.part;
    shm_zone = part->elts;

    // 遍历所有共享内存区域
    for (i = 0; /* void */; i++) {

        if (i >= part->nelts) {
            if (part->next == NULL) {
                break;
            }
            part = part->next;
            shm_zone = part->elts;
            i = 0;
        }

        // 分配共享内存
        if (ngx_shm_alloc(&shm_zone[i].shm) != NGX_OK) {
            goto failed;
        }

        // 初始化共享内存区域
        if (ngx_init_zone_pool(cycle, &shm_zone[i]) != NGX_OK) {
            goto failed;
        }

        // 调用模块特定的初始化函数
        if (shm_zone[i].init(&shm_zone[i], old_shm_zone[i].data) != NGX_OK) {
            goto failed;
        }
    }

    return cycle;

failed:
    ngx_destroy_cycle_pools(&conf);
    return NULL;
}
```

### 6.2 Worker进程中的共享内存继承

Worker进程通过fork继承master进程的共享内存映射：

```c
// Worker进程启动函数
static void ngx_worker_process_cycle(ngx_cycle_t *cycle, void *data)
{
    ngx_int_t worker = (intptr_t) data;

    ngx_process = NGX_PROCESS_WORKER;
    ngx_worker = worker;

    // Worker进程继承了master进程的内存映射
    // 共享内存区域在fork后自动可用

    // 初始化worker进程特定的事件处理
    if (ngx_event_process_init(cycle) != NGX_OK) {
        exit(2);
    }

    // 进入worker进程主循环
    ngx_worker_process_main(cycle);
}
```

## 7. 实际应用示例

### 7.1 SSL会话缓存

SSL会话缓存使用共享内存在worker进程间共享SSL会话信息：

```c
// SSL会话缓存的共享内存使用示例
typedef struct {
    ngx_rbtree_t       rbtree;      // 红黑树存储会话
    ngx_rbtree_node_t  sentinel;    // 哨兵节点
    ngx_queue_t        expire_queue; // 过期队列
    ngx_uint_t         current;     // 当前会话数
    ngx_uint_t         max;         // 最大会话数
} ngx_ssl_session_cache_t;

// 从共享内存中获取SSL会话
SSL_SESSION *ngx_ssl_get_cached_session(ngx_ssl_conn_t *ssl_conn,
    u_char *id, int len, int *copy)
{
    ngx_ssl_session_cache_t  *cache;
    ngx_ssl_session_node_t   *sess_node;

    // 获取共享内存中的缓存
    cache = (ngx_ssl_session_cache_t *) ssl_conn->session_ctx->data;

    // 在红黑树中查找会话
    sess_node = ngx_ssl_session_lookup(cache, id, len);

    if (sess_node) {
        // 更新访问时间
        ngx_queue_remove(&sess_node->queue);
        ngx_queue_insert_head(&cache->expire_queue, &sess_node->queue);

        *copy = 0;  // 不需要复制，直接返回共享内存中的会话
        return sess_node->session;
    }

    return NULL;
}
```

### 7.2 统计信息收集

共享内存还用于收集和共享各种统计信息：

```c
// 全局统计信息结构
typedef struct {
    ngx_atomic_t  requests;          // 总请求数
    ngx_atomic_t  responses_2xx;     // 2xx响应数
    ngx_atomic_t  responses_4xx;     // 4xx响应数
    ngx_atomic_t  responses_5xx;     // 5xx响应数
    ngx_atomic_t  bytes_sent;        // 发送字节数
    ngx_atomic_t  bytes_received;    // 接收字节数
} ngx_http_stats_t;

// 更新统计信息（原子操作）
static void ngx_http_update_stats(ngx_http_request_t *r)
{
    ngx_http_stats_t *stats;

    // 获取共享内存中的统计结构
    stats = (ngx_http_stats_t *) ngx_shared_stats->data;

    // 原子递增请求计数
    (void) ngx_atomic_fetch_add(&stats->requests, 1);

    // 根据响应状态更新相应计数器
    if (r->headers_out.status >= 200 && r->headers_out.status < 300) {
        (void) ngx_atomic_fetch_add(&stats->responses_2xx, 1);
    } else if (r->headers_out.status >= 400 && r->headers_out.status < 500) {
        (void) ngx_atomic_fetch_add(&stats->responses_4xx, 1);
    } else if (r->headers_out.status >= 500) {
        (void) ngx_atomic_fetch_add(&stats->responses_5xx, 1);
    }

    // 更新字节统计
    (void) ngx_atomic_fetch_add(&stats->bytes_sent, r->connection->sent);
}
```

## 8. 总结

nginx的共享内存实现是其高性能多进程架构的关键组成部分。通过精心设计的数据结构和同步机制，nginx实现了：

1. **高效的内存管理**：通过slab分配器实现快速的内存分配和释放
2. **可靠的同步机制**：使用原子操作和互斥锁确保数据一致性
3. **灵活的应用框架**：支持各种模块的共享内存需求
4. **跨平台兼容性**：针对不同操作系统提供相应的实现

这种设计使得nginx能够在保持高性能的同时，实现复杂的功能如限流、负载均衡、会话管理等，是现代高性能web服务器架构的典型代表。

理解nginx共享内存的实现原理，对于深入掌握nginx架构、进行性能优化以及开发自定义模块都具有重要意义。
```
```
