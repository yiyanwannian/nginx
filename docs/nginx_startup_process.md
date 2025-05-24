# NGINX 启动流程详解

本文档详细分析了 NGINX 的启动流程，从程序入口到主循环的建立，帮助开发者理解 NGINX 的工作原理。

## 1. 程序入口

NGINX 的程序入口位于 `src/core/nginx.c` 文件中的 `main()` 函数。启动过程主要包括以下几个阶段：

### 1.1 初始化基础组件

```c
int ngx_cdecl
main(int argc, char *const *argv)
{
    ngx_buf_t        *b;
    ngx_log_t        *log;
    ngx_uint_t        i;
    ngx_cycle_t      *cycle, init_cycle;
    ngx_conf_dump_t  *cd;
    ngx_core_conf_t  *ccf;

    // 初始化调试功能
    ngx_debug_init();

    // 初始化错误字符串处理
    if (ngx_strerror_init() != NGX_OK) {
        return 1;
    }

    // 处理命令行选项
    if (ngx_get_options(argc, argv) != NGX_OK) {
        return 1;
    }

    // 显示版本信息（如果需要）
    if (ngx_show_version) {
        ngx_show_version_info();
        if (!ngx_test_config) {
            return 0;
        }
    }

    // 初始化时间
    ngx_time_init();

    // 初始化正则表达式库（如果编译时包含）
    #if (NGX_PCRE)
        ngx_regex_init();
    #endif

    // 获取进程ID
    ngx_pid = ngx_getpid();
    ngx_parent = ngx_getppid();

    // 初始化日志系统
    log = ngx_log_init(ngx_prefix, ngx_error_log);
    if (log == NULL) {
        return 1;
    }

    // 初始化SSL（如果编译时包含）
    #if (NGX_OPENSSL)
        ngx_ssl_init(log);
    #endif
```

### 1.2 初始化周期结构

NGINX 使用 `ngx_cycle_t` 结构体来存储运行时的各种数据，包括配置、连接、事件等。初始化过程如下：

```c
    // 初始化周期结构
    ngx_memzero(&init_cycle, sizeof(ngx_cycle_t));
    init_cycle.log = log;
    ngx_cycle = &init_cycle;

    // 创建内存池
    init_cycle.pool = ngx_create_pool(1024, log);
    if (init_cycle.pool == NULL) {
        return 1;
    }

    // 保存命令行参数
    if (ngx_save_argv(&init_cycle, argc, argv) != NGX_OK) {
        return 1;
    }

    // 处理命令行选项
    if (ngx_process_options(&init_cycle) != NGX_OK) {
        return 1;
    }

    // 初始化操作系统相关功能
    if (ngx_os_init(log) != NGX_OK) {
        return 1;
    }

    // 初始化CRC32表
    if (ngx_crc32_table_init() != NGX_OK) {
        return 1;
    }

    // 初始化内存分配相关参数
    ngx_slab_sizes_init();

    // 添加继承的套接字（如果有）
    if (ngx_add_inherited_sockets(&init_cycle) != NGX_OK) {
        return 1;
    }

    // 预初始化模块
    if (ngx_preinit_modules() != NGX_OK) {
        return 1;
    }
```

### 1.3 初始化完整周期

```c
    // 初始化完整的NGINX周期
    cycle = ngx_init_cycle(&init_cycle);
    if (cycle == NULL) {
        if (ngx_test_config) {
            ngx_log_stderr(0, "configuration file %s test failed",
                           init_cycle.conf_file.data);
        }
        return 1;
    }

    // 如果只是测试配置，则退出
    if (ngx_test_config) {
        if (!ngx_quiet_mode) {
            ngx_log_stderr(0, "configuration file %s test is successful",
                           cycle->conf_file.data);
        }
        return 0;
    }

    // 处理信号
    if (ngx_signal) {
        return ngx_signal_process(cycle, ngx_signal);
    }
```

### 1.4 启动主循环

根据配置决定以单进程模式或主进程/工作进程模式运行：

```c
    // 获取核心配置
    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);

    // 确定进程模式
    if (ccf->master && ngx_process == NGX_PROCESS_SINGLE) {
        ngx_process = NGX_PROCESS_MASTER;
    }

    // 初始化信号处理
    if (ngx_init_signals(cycle->log) != NGX_OK) {
        return 1;
    }

    // 如果配置为守护进程模式，则转为后台运行
    if (!ngx_inherited && ccf->daemon) {
        if (ngx_daemon(cycle->log) != NGX_OK) {
            return 1;
        }
        ngx_daemonized = 1;
    }

    // 创建PID文件
    if (ngx_create_pidfile(&ccf->pid, cycle->log) != NGX_OK) {
        return 1;
    }

    // 根据进程模式启动相应的主循环
    if (ngx_process == NGX_PROCESS_SINGLE) {
        ngx_single_process_cycle(cycle);
    } else {
        ngx_master_process_cycle(cycle);
    }

    return 0;
}
```

## 2. 周期初始化详解

`ngx_init_cycle()` 函数是 NGINX 启动过程中的核心部分，负责初始化配置、模块、共享内存等。

```c
ngx_cycle_t *
ngx_init_cycle(ngx_cycle_t *old_cycle)
{
    void                *rv, *data;
    char               **senv;
    ngx_uint_t           i, n;
    ngx_log_t           *log;
    ngx_time_t          *tp;
    ngx_conf_t           conf;
    ngx_pool_t          *pool;
    ngx_cycle_t         *cycle, **old;
    ngx_shm_zone_t      *shm_zone, *oshm_zone;
    ngx_list_part_t     *part, *opart;
    ngx_open_file_t     *file;
    ngx_listening_t     *ls, *nls;
    ngx_core_conf_t     *ccf, *old_ccf;
    ngx_core_module_t   *module;
    char                 hostname[NGX_MAXHOSTNAMELEN];

    // 更新时区和时间
    ngx_timezone_update();
    tp = ngx_timeofday();
    tp->sec = 0;
    ngx_time_update();

    // 获取日志对象
    log = old_cycle->log;

    // 创建内存池
    pool = ngx_create_pool(NGX_CYCLE_POOL_SIZE, log);
    if (pool == NULL) {
        return NULL;
    }
    pool->log = log;

    // 分配新的周期结构
    cycle = ngx_pcalloc(pool, sizeof(ngx_cycle_t));
    if (cycle == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }

    // 初始化周期结构
    cycle->pool = pool;
    cycle->log = log;
    cycle->old_cycle = old_cycle;
    // ... 复制路径和配置参数 ...

    // 初始化各种数组和队列
    // ... 初始化paths, config_dump, listening等数组 ...

    // 初始化可重用连接队列
    ngx_queue_init(&cycle->reusable_connections_queue);

    // 分配模块配置上下文数组
    cycle->conf_ctx = ngx_pcalloc(pool, ngx_max_module * sizeof(void *));
    if (cycle->conf_ctx == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }

    // 获取主机名
    if (gethostname(hostname, NGX_MAXHOSTNAMELEN) == -1) {
        ngx_log_error(NGX_LOG_EMERG, log, ngx_errno, "gethostname() failed");
        ngx_destroy_pool(pool);
        return NULL;
    }
    // ... 处理主机名 ...

    // 初始化模块
    if (ngx_cycle_modules(cycle) != NGX_OK) {
        ngx_destroy_pool(pool);
        return NULL;
    }

    // 创建每个核心模块的配置
    for (i = 0; cycle->modules[i]; i++) {
        if (cycle->modules[i]->type != NGX_CORE_MODULE) {
            continue;
        }

        module = cycle->modules[i]->ctx;

        if (module->create_conf) {
            rv = module->create_conf(cycle);
            if (rv == NULL) {
                ngx_destroy_pool(pool);
                return NULL;
            }
            cycle->conf_ctx[cycle->modules[i]->index] = rv;
        }
    }

    // 保存环境变量
    senv = environ;

    // 初始化配置解析器
    ngx_memzero(&conf, sizeof(ngx_conf_t));
    conf.args = ngx_array_create(pool, 10, sizeof(ngx_str_t));
    if (conf.args == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }

    conf.temp_pool = ngx_create_pool(NGX_CYCLE_POOL_SIZE, log);
    if (conf.temp_pool == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }

    conf.ctx = cycle->conf_ctx;
    conf.cycle = cycle;
    conf.pool = pool;
    conf.log = log;
    conf.module_type = NGX_CORE_MODULE;
    conf.cmd_type = NGX_MAIN_CONF;

    // 解析命令行参数中的配置
    if (ngx_conf_param(&conf) != NGX_CONF_OK) {
        environ = senv;
        ngx_destroy_cycle_pools(&conf);
        return NULL;
    }

    // 解析配置文件
    if (ngx_conf_parse(&conf, &cycle->conf_file) != NGX_CONF_OK) {
        environ = senv;
        ngx_destroy_cycle_pools(&conf);
        return NULL;
    }

    // 初始化每个核心模块的配置
    for (i = 0; cycle->modules[i]; i++) {
        if (cycle->modules[i]->type != NGX_CORE_MODULE) {
            continue;
        }

        module = cycle->modules[i]->ctx;

        if (module->init_conf) {
            if (module->init_conf(cycle, cycle->conf_ctx[cycle->modules[i]->index])
                == NGX_CONF_ERROR)
            {
                environ = senv;
                ngx_destroy_cycle_pools(&conf);
                return NULL;
            }
        }
    }

    // ... 处理PID文件 ...

    // ... 初始化共享内存 ...

    // 打开监听套接字
    if (ngx_open_listening_sockets(cycle) != NGX_OK) {
        goto failed;
    }

    // 配置监听套接字
    if (!ngx_test_config) {
        ngx_configure_listening_sockets(cycle);
    }

    // 初始化模块
    if (ngx_init_modules(cycle) != NGX_OK) {
        exit(1);
    }

    // ... 清理旧周期 ...

    // 返回初始化完成的周期
    return cycle;

failed:
    // ... 错误处理 ...
    return NULL;
}
```

## 3. 主进程循环

NGINX 支持两种运行模式：单进程模式和主进程/工作进程模式。在生产环境中，通常使用主进程/工作进程模式。

### 3.1 主进程循环 (Master Process Cycle)

主进程负责管理工作进程和处理信号：

```c
void
ngx_master_process_cycle(ngx_cycle_t *cycle)
{
    char              *title;
    u_char            *p;
    size_t             size;
    ngx_int_t          i;
    ngx_uint_t         sigio;
    sigset_t           set;
    struct itimerval   itv;
    ngx_uint_t         live;
    ngx_msec_t         delay;
    ngx_core_conf_t   *ccf;

    // 初始化信号集，添加需要处理的信号
    sigemptyset(&set);
    sigaddset(&set, SIGCHLD); // 子进程状态改变信号
    sigaddset(&set, SIGALRM); // 定时器信号
    sigaddset(&set, SIGIO);   // I/O信号
    sigaddset(&set, SIGINT);  // 中断信号
    sigaddset(&set, ngx_signal_value(NGX_RECONFIGURE_SIGNAL)); // 重新加载配置
    sigaddset(&set, ngx_signal_value(NGX_REOPEN_SIGNAL));      // 重新打开日志
    sigaddset(&set, ngx_signal_value(NGX_NOACCEPT_SIGNAL));    // 停止接受新连接
    sigaddset(&set, ngx_signal_value(NGX_TERMINATE_SIGNAL));   // 终止信号
    sigaddset(&set, ngx_signal_value(NGX_SHUTDOWN_SIGNAL));    // 优雅关闭信号
    sigaddset(&set, ngx_signal_value(NGX_CHANGEBIN_SIGNAL));   // 切换二进制信号

    // 阻塞上述信号，避免在主循环中被中断
    if (sigprocmask(SIG_BLOCK, &set, NULL) == -1) {
        ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                      "sigprocmask() failed");
    }

    sigemptyset(&set); // 清空信号集

    // 设置进程标题
    // ... 设置进程标题代码 ...

    // 获取核心配置
    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);

    // 启动工作进程和缓存管理进程
    ngx_start_worker_processes(cycle, ccf->worker_processes, NGX_PROCESS_RESPAWN);
    ngx_start_cache_manager_processes(cycle, 0);

    // 主循环变量初始化
    ngx_new_binary = 0;
    delay = 0;
    sigio = 0;
    live = 1;

    // 主循环，处理信号和子进程管理
    for ( ;; ) {
        // 设置定时器（如果需要）
        if (delay) {
            // ... 设置定时器代码 ...
        }

        // 等待信号
        sigsuspend(&set);

        // 更新时间
        ngx_time_update();

        // 处理子进程退出
        if (ngx_reap) {
            ngx_reap = 0;
            live = ngx_reap_children(cycle);
            if (!live && (ngx_terminate || ngx_quit)) {
                ngx_master_process_exit(cycle);
            }
        }

        // 处理终止信号
        if (ngx_terminate) {
            // ... 终止处理代码 ...
            continue;
        }

        // 处理优雅退出信号
        if (ngx_quit) {
            // ... 优雅退出处理代码 ...
            continue;
        }

        // 处理重新配置信号
        if (ngx_reconfigure) {
            // ... 重新配置处理代码 ...
            continue;
        }

        // 处理重启信号
        if (ngx_restart) {
            // ... 重启处理代码 ...
            continue;
        }

        // 处理重新打开日志信号
        if (ngx_reopen) {
            // ... 重新打开日志处理代码 ...
            continue;
        }

        // 处理切换二进制文件信号
        if (ngx_change_binary) {
            // ... 切换二进制文件处理代码 ...
            continue;
        }

        // 处理停止接受连接信号
        if (ngx_noaccept) {
            // ... 停止接受连接处理代码 ...
            continue;
        }
    }
}
```

### 3.2 工作进程启动

主进程通过 `ngx_start_worker_processes()` 函数启动工作进程：

```c
static ngx_int_t
ngx_start_worker_processes(ngx_cycle_t *cycle, ngx_int_t n, ngx_int_t type)
{
    ngx_int_t      i;
    ngx_channel_t  ch;

    ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "start worker processes");

    ngx_memzero(&ch, sizeof(ngx_channel_t));

    ch.command = NGX_CMD_OPEN_CHANNEL;

    for (i = 0; i < n; i++) {
        // 生成工作进程
        ngx_spawn_process(cycle, ngx_worker_process_cycle,
                          (void *) (intptr_t) i, "worker process", type);

        ch.pid = ngx_processes[ngx_process_slot].pid;
        ch.slot = ngx_process_slot;
        ch.fd = ngx_processes[ngx_process_slot].channel[0];

        // 为每个工作进程打开通道
        ngx_pass_open_channel(cycle, &ch);
    }

    return NGX_OK;
}
```

### 3.3 工作进程循环

工作进程通过 `ngx_worker_process_cycle()` 函数运行其主循环：

```c
static void
ngx_worker_process_cycle(ngx_cycle_t *cycle, void *data)
{
    ngx_int_t worker = (intptr_t) data;

    // ... 初始化工作进程 ...

    // 工作进程主循环
    for ( ;; ) {
        // 如果正在退出且没有定时器，则退出循环
        if (ngx_exiting) {
            if (ngx_event_no_timers_left() == NGX_OK) {
                break;
            }
        }

        // 处理事件和定时器
        ngx_process_events_and_timers(cycle);

        // 处理各种信号
        if (ngx_terminate) {
            return;
        }

        if (ngx_quit) {
            // ... 优雅退出处理 ...
        }

        if (ngx_reopen) {
            // ... 重新打开文件处理 ...
        }
    }

    // 退出工作进程
    ngx_worker_process_exit(cycle);
}
```

## 4. 单进程循环

在开发和调试环境中，可以使用单进程模式运行 NGINX：

```c
void
ngx_single_process_cycle(ngx_cycle_t *cycle)
{
    ngx_uint_t  i;

    // 初始化信号处理
    if (ngx_set_environment(cycle, NULL) == NULL) {
        /* fatal */
        exit(2);
    }

    // 初始化模块
    for (i = 0; cycle->modules[i]; i++) {
        if (cycle->modules[i]->init_process) {
            if (cycle->modules[i]->init_process(cycle) == NGX_ERROR) {
                /* fatal */
                exit(2);
            }
        }
    }

    // 单进程主循环
    for ( ;; ) {
        // 处理事件和定时器
        ngx_process_events_and_timers(cycle);

        // 处理各种信号
        if (ngx_terminate || ngx_quit) {
            break;
        }

        if (ngx_reopen) {
            // ... 重新打开文件处理 ...
        }
    }

    // 退出前清理
    for (i = 0; cycle->modules[i]; i++) {
        if (cycle->modules[i]->exit_process) {
            cycle->modules[i]->exit_process(cycle);
        }
    }
}
```

## 5. 启动流程总结

NGINX 的启动流程可以总结为以下几个主要步骤：

1. **初始化基础组件**：
   - 处理命令行选项
   - 初始化时间、日志、正则表达式等基础组件

2. **创建和初始化周期结构**：
   - 分配内存池
   - 初始化各种数组和队列
   - 加载模块

3. **解析配置**：
   - 解析命令行参数中的配置
   - 解析配置文件
   - 初始化模块配置

4. **打开监听套接字**：
   - 创建和配置监听套接字
   - 准备接受连接

5. **启动主循环**：
   - 根据配置选择单进程模式或主进程/工作进程模式
   - 在主进程/工作进程模式下，主进程负责管理工作进程和处理信号
   - 工作进程负责处理连接和请求

6. **处理信号和事件**：
   - 主进程处理各种信号（如重新加载配置、重启、退出等）
   - 工作进程处理网络事件和定时器事件

通过这种设计，NGINX 实现了高性能、高可靠性和高可扩展性，能够处理大量并发连接，并支持平滑升级和配置重载。
