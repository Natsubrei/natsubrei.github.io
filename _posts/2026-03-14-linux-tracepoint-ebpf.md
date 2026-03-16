---
layout: post
title: "Linux Tracepoint 与 eBPF 深度解析"
date: 2026-03-14 10:00:00
categories: [技术笔记]
tags: [linux, tracepoint, ebpf, kernel]
---

# Linux Tracepoint 与 eBPF 深度解析

> 本文基于 Linux 6.19.8 内核源码分析

## 引言

在 Linux 性能分析和调试中，tracepoint 和 eBPF 是两项非常重要的技术。本文通过剖析这两个概念及其关系，帮助读者深入理解内核的跟踪机制。

---

## 1. tracepoint 基础

### 1.1 什么是 tracepoint？

tracepoint 可以理解为内核代码中的**埋点**。在特定位置插入一段代码，当程序执行到这里时，会检查是否有"探针"注册，如果有就调用这些探针函数。

```
普通代码执行:
    do_something() → do_otherthing()

有 tracepoint 的执行:
    do_something() → trace_xxx() → 如果有 probe 就调用 → do_otherthing()
```

### 1.2 一个具体的例子：sched_switch

`sched_switch` 是 Linux 内核中最重要的 tracepoint 之一，用于记录**任务调度切换**事件。每次 CPU 从一个进程切换到另一个进程时，这个点都会被触发。

#### 触发时机

- 时间片耗尽，进程被抢占
- 进程主动调用 `schedule()` 让出 CPU
- 进程阻塞等待 I/O、锁等资源
- 高优先级进程被唤醒，需要抢占
- 中断或系统调用返回时触发调度

#### 定义方式

在 `include/trace/events/sched.h` 中，`sched_switch` 是这样定义的：

```c
TRACE_EVENT(sched_switch,

    TP_PROTO(bool preempt,
             struct task_struct *prev,
             struct task_struct *next,
             unsigned int prev_state),

    TP_ARGS(preempt, prev, next, prev_state),

    TP_STRUCT__entry(
        __array(char,  prev_comm,  TASK_COMM_LEN)
        __field(pid_t, prev_pid)
        __field(int,   prev_prio)
        __field(long,  prev_state)
        __array(char,  next_comm,  TASK_COMM_LEN)
        __field(pid_t, next_pid)
        __field(int,   next_prio)
    ),

    TP_fast_assign(
        memcpy(__entry->prev_comm, prev->comm, TASK_COMM_LEN);
        __entry->prev_pid   = prev->pid;
        __entry->prev_prio  = prev->prio;
        __entry->prev_state = __trace_sched_switch_state(preempt, prev_state, prev);
        memcpy(__entry->next_comm, next->comm, TASK_COMM_LEN);
        __entry->next_pid   = next->pid;
        __entry->next_prio  = next->prio;
    ),

    TP_printk("prev_comm=%s prev_pid=%d prev_prio=%d prev_state=%s ==> "
              "next_comm=%s next_pid=%d next_prio=%d",
        __entry->prev_comm, __entry->prev_pid, __entry->prev_prio,
        /* 状态格式化 */,
        __entry->next_comm, __entry->next_pid, __entry->next_prio)
);
```

这个定义声明了一个 tracepoint，记录：
- **prev**: 被换下的进程（名称、PID、优先级、状态）
- **next**: 换上的进程（名称、PID、优先级）

---

## 2. tracepoint 的实现原理

### 2.1 数据结构

tracepoint 的核心数据结构定义在 `include/linux/tracepoint-defs.h`：

```c
// include/linux/tracepoint-defs.h
struct tracepoint {
    const char *name;                    // tracepoint 名称
    struct static_key_false key;         // 启用/关闭开关
    struct tracepoint_func __rcu *funcs; // 注册的 probe 列表
    // ...
};

struct tracepoint_func {
    void *func;                          // probe 函数指针
    void *data;                          // 传递给 probe 的数据
    int prio;                            // 优先级
    // ...
};
```

### 2.2 条件编译：stub 机制

在内核代码中，tracepoint 通过条件编译来实现。当没有定义 `CREATE_TRACE_POINTS` 时，tracepoint 只是一个空函数：

```c
// include/linux/tracepoint.h
#define __DECLARE_TRACE_COMMON(name, proto, args, data_proto)    \
    static inline void trace_##name(proto) { }    /* 空实现 */ \
    static inline int                                      \
    register_trace_##name(void (*probe)(data_proto), void *data) \
    { return -ENOSYS; }    /* 返回"未实现" */           \
    static inline bool                                     \
    trace_##name##_enabled(void) { return false; }
```

当定义了 `CREATE_TRACE_POINTS` 时，会生成真正的实现。

### 2.3 运行时开关：static_branch

真正的实现使用 `static_branch_unlikely` 来实现条件判断：

```c
// include/linux/tracepoint.h
static inline void trace_##name(proto)
{
    // 检查 key 标志位
    if (static_branch_unlikely(&__tracepoint_##name.key))
        __do_trace_##name(args);
}
```

当没有任何 probe 注册时，`key` 默认是"关闭"状态，CPU 可以快速跳过检查，几乎零开销。

### 2.4 注册与启用

当第一个 probe 被注册时，会调用 `static_branch_enable(&tp->key)` 启用这个 key：

```c
// kernel/trace/tracepoint.c
static int tracepoint_add_func(struct tracepoint *tp,
               struct tracepoint_func *func, int prio, bool warn)
{
    struct tracepoint_func *old, *tp_funcs;
    // ...
    old = func_add(&tp_funcs, func, prio);
    // ...
    switch (nr_func_state(tp_funcs)) {
    case TP_FUNC_1:        /* 0->1 */
        // ...
        static_branch_enable(&tp->key);  // 关键！
        break;
    // ...
    }
}
```

---

## 3. tracepoint 的触发与注册

### 3.1 调度器中的触发

`sched_switch` 在调度器的核心函数 `__schedule()` 中被调用：

```c
// kernel/sched/core.c
static void __sched notrace __schedule(int sched_mode)
{
    // ... 选择新进程，处理调度逻辑 ...

    trace_sched_switch(preempt, prev, next, prev_state);  // 触发

    rq = context_switch(rq, prev, next, &rf);  // 执行实际切换
}
```

### 3.2 触发时的完整流程

```
schedule()                              // 调度入口
    ↓
__schedule_loop()                      // 调度循环
    ↓
__schedule(int sched_mode)             // 调度核心
    ↓
选择新进程
    ↓
trace_sched_switch(...)                // 触发 tracepoint
    ↓
static_branch_unlikely(&key)           // 检查是否开启
    ↓ (开启时)
遍历调用所有已注册的 probe 函数
```

### 3.3 什么是注册？

注册就是告诉内核："当这个 tracepoint 被触发时，请调用我的函数"。

```c
// 内核模块中的例子
void my_probe(void *data, bool preempt,
    struct task_struct *prev, struct task_struct *next,
    unsigned int prev_state)
{
    printk("进程切换: %s -> %s\n", prev->comm, next->comm);
}

// 注册这个 probe
register_trace_sched_switch(my_probe, NULL);
```

---

## 4. 用户空间如何使用 tracepoint？

tracepoint 可以在用户空间使用，**不需要自己写内核模块**。有两种主要方式：

### 4.1 ftrace

ftrace 是内核内置的跟踪工具，直接使用即可：

```bash
# 挂载 debugfs
mount -t debugfs none /sys/kernel/debug

# 开启跟踪
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo nop > current_tracer
echo 1 > events/sched/sched_switch/enable
echo 1 > tracing_on

# 查看输出
cat trace

# 关闭
echo 0 > events/sched/sched_switch/enable
```

### 4.2 perf

```bash
perf record -e sched:sched_switch -a -g sleep 5
perf report
```

### 4.3 ftrace/perf 的实现原理

ftrace 和 perf 内部已经写好了 probe，当用户开启跟踪时，它们会调用 `register_trace_sched_switch()` 注册自己的 probe：

```
用户: echo 1 > enable
    ↓
ftrace/perf 内核代码调用:
    register_trace_sched_switch(ftrace_probe, ...)
    ↓
sched_switch 触发时:
    trace_sched_switch() → ftrace_probe() → 写入 ring buffer
    ↓
用户: cat trace → 读取 ring buffer
```

### 4.4 为什么有了 tracepoint 还需要 ftrace 和 perf？

tracepoint 提供了底层机制，但光有 tracepoint 没法直接用。

#### tracepoint 提供了什么？

- 埋点位置
- 注册/注销 probe 的接口
- 触发时调用 probe 的机制

#### tracepoint 没有提供什么？

| 缺失的功能 | 说明 |
|-----------|------|
| 有哪些 tracepoint 可用？ | 不知道 |
| 如何开启/关闭？ | 不知道怎么操作 |
| 数据存哪里？ | 没有存储 |
| 如何过滤？ | 没有过滤功能 |
| 如何查看结果？ | 没有 UI |

#### ftrace 提供了什么？

基于 tracepoint 实现了：

1. **文件系统接口** - `/sys/kernel/debug/tracing/`
2. **tracepoint 管理** - 列出所有可用的 tracepoint，一键开启/关闭
3. **数据存储** - ring buffer，存放大量的跟踪数据
4. **多种跟踪器** - function、function_graph、hwlat 等
5. **过滤功能** - 按进程、函数名过滤

#### perf 提供了什么？

1. **硬件性能计数器** - CPU 周期、缓存命中等
2. **采样功能** - 周期性采样，不影响性能
3. **统计分析** - 生成报告，如热点分析
4. **绑定 tracepoint** - 也可以用 tracepoint

### 4.5 perf 的数据来源

perf 不仅可以使用 tracepoint，还支持多种数据来源：

| 数据来源 | 例子 |
|---------|------|
| tracepoint | sched_switch |
| kprobe | 内核函数入口 |
| uprobe | 用户函数入口 |
| hardware | CPU 周期、缓存命中 |
| software | 上下文切换、缺页异常 |

### 4.6 总结：层级关系

```
tracepoint = 底层机制（提供埋点、触发、注册接口）
    ↑
ftrace = 基于 tracepoint
perf   = 基于 tracepoint + 其他多种数据源
```

**简单说**：tracepoint 是"怎么实现"，ftrace/perf 是"怎么使用"。光有 tracepoint 没法直接用，需要工具来操作它。

---

## 5. sysfs/debugfs 接口

tracepoint 可以通过文件系统动态控制，这是 ftrace、perf 等工具工作的基础。

`/sys` 和 `/sys/kernel/debug` 是 Linux 内核提供的虚拟文件系统，用于向用户空间暴露内核状态和控制接口。

### 5.1 控制流程

```
echo 1 > /sys/kernel/debug/tracing/events/sched/sched_switch/enable
    ↓
VFS write 系统调用
    ↓
event_enable_write()           // kernel/trace/trace_events.c
    ↓
ftrace_event_enable_disable()  // kernel/trace/trace_events.c
    ↓
tracepoint_probe_register()    // kernel/trace/tracepoint.c
    ↓
static_branch_enable(&key)    // 启用 tracepoint
```

---

## 6. eBPF 是什么？

### 6.1 概念

**eBPF** (Extended Berkeley Packet Filter) 是 Linux 内核中的一个革命性机制。它允许你在内核中**安全地运行自定义程序**，而无需修改内核代码或加载内核模块。

### 6.2 eBPF 与 tracepoint 的关系

```
tracepoint:    内核埋的固定钩子 → 你只能被动接收
eBPF:          你可以主动编写程序 → 挂载到 tracepoint 等各种钩子上
```

eBPF 程序可以挂载到：
- tracepoint
- kprobe（内核函数入口）
- uprobe（用户函数入口）
- 网络包（packet socket）
- 性能事件（perf_event）

### 6.3 为什么需要 eBPF？

| 方式 | 灵活性 |
|------|--------|
| ftrace/perf | 工具内置功能，不能自定义 |
| eBPF | 随便写程序，想怎么处理都行 |

---

## 7. eBPF 的工作原理

### 7.1 核心架构

```
┌─────────────────────────────────────────────────────────────┐
│                      用户空间                               │
│                                                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐   │
│  │ eBPF 程序   │ →  │ clang 编译  │ →  │ 字节码      │   │
│  │ (C/Rust)   │    │             │    │ (.bpf.o)    │   │
│  └─────────────┘    └─────────────┘    └──────┬──────┘   │
│                                               │           │
│                                    bpf() 系统调用          │
└───────────────────────────────────────────────┼───────────┘
                                                ↓
┌─────────────────────────────────────────────────────────────┐
│                      内核空间                               │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    BPF Subsystem                     │   │
│  │                                                      │   │
│  │  1. bpf() syscall → 接收字节码                     │   │
│  │           ↓                                         │   │
│  │  2. Verifier → 检查安全性                           │   │
│  │           ↓                                         │   │
│  │  3. JIT 编译 → 机器码 (可选)                       │   │
│  │           ↓                                         │   │
│  │  4. 附加到 hook 点                                 │   │
│  │           ↓                                         │   │
│  │  5. 事件触发 → 执行程序                            │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 bpf() 系统调用入口

```c
// kernel/bpf/syscall.c
SYSCALL_DEFINE3(bpf, ...)
{
    return __sys_bpf(...);
}
```

支持的命令：
- `BPF_MAP_CREATE` - 创建 eBPF map
- `BPF_PROG_LOAD` - 加载 eBPF 程序
- `BPF_RAW_TRACEPOINT_OPEN` - 附加到 tracepoint

### 7.3 验证：Verifier

这是 eBPF 安全的核心。Verifier 会检查程序是否：
- 访问合法内存
- 没有无限制循环
- 不访问内核私有数据

```c
static int do_check(struct bpf_verifier_env *env)
{
    // ...
}
```

### 7.4 JIT 编译

验证通过后，可以 JIT 编译成机器码，执行效率和原生内核代码一样快。

---

## 8. eBPF 挂载到 tracepoint 的代码流程

### 8.1 用户空间操作

```c
// 用户空间
bpf(BPF_PROG_LOAD, &attr, sizeof(attr));    // 加载 eBPF 程序
bpf(BPF_RAW_TRACEPOINT_OPEN, &attr, ...);    // 附加到 tracepoint
```

### 8.2 内核处理流程

**步骤 1**: bpf() 系统调用接收命令

```c
// kernel/bpf/syscall.c
SYSCALL_DEFINE3(bpf, ...)
{
    return __sys_bpf(...);
}
```

**步骤 2**: 处理 RAW_TRACEPOINT_OPEN 命令

```c
// kernel/bpf/syscall.c
case BPF_RAW_TRACEPOINT_OPEN:
    err = bpf_raw_tracepoint_open(&attr);
```

**步骤 3**: 注册到 tracepoint

```c
// kernel/bpf/syscall.c
err = bpf_probe_register(link->btp, link);
```

**步骤 4**: 调用内核的 tracepoint 注册函数

```c
// kernel/trace/bpf_trace.c
int bpf_probe_register(struct bpf_raw_event_map *btp, struct bpf_raw_tp_link *link)
{
    struct tracepoint *tp = btp->tp;
    struct bpf_prog *prog = link->link.prog;

    // 检查程序访问的参数是否超过 tracepoint 可用的参数
    if (prog->aux->max_ctx_offset > btp->num_args * sizeof(u64))
        return -EINVAL;
    // ...
    return tracepoint_probe_register_may_exist(tp, (void *)btp->bpf_func, link);
}
```

**关键点**：这里调用了 `tracepoint_probe_register_may_exist()`，这是内核通用的 tracepoint 注册函数。**eBPF 和 ftrace/perf 本质一样，都是调用同一个注册函数**。

### 8.3 触发时的执行流程

当 `sched_switch` 触发时：

```c
// kernel/sched/core.c
trace_sched_switch(preempt, prev, next, prev_state);
```

→ tracepoint 内部遍历调用所有 probe：

```c
// include/linux/tracepoint.h
if (static_branch_unlikely(&__tracepoint_sched_switch.key))
    __do_trace_sched_switch(args);
```

→ 调用 eBPF 程序：

```c
// kernel/trace/bpf_trace.c
void __bpf_trace_run(struct bpf_raw_tp_link *link, u64 *args)
{
    struct bpf_prog *prog = link->link.prog;
    struct bpf_run_ctx *old_run_ctx;
    struct bpf_trace_run_ctx run_ctx;

    cant_sleep();
    if (unlikely(this_cpu_inc_return(*(prog->active)) != 1)) {
        bpf_prog_inc_misses_counter(prog);
        goto out;
    }

    run_ctx.bpf_cookie = link->cookie;
    old_run_ctx = bpf_set_run_ctx(&run_ctx.run_ctx);

    rcu_read_lock();
    (void) bpf_prog_run(prog, args);
    rcu_read_unlock();

    bpf_reset_run_ctx(old_run_ctx);
out:
    this_cpu_dec(*(prog->active));
}
```

### 8.4 为什么需要中间层？

你可能会问：为什么 eBPF 程序不能直接作为 tracepoint 的 probe，而需要通过 `__bpf_trace_run` 这个"包装函数"？

原因如下：

1. **eBPF 程序需要 JIT 编译** - tracepoint 的 probe 需要直接可执行的机器码，而 eBPF 程序是字节码，需要先经过 JIT 编译

2. **执行上下文不同** - eBPF 有自己的上下文管理机制，需要设置 `bpf_run_ctx`

3. **安全性** - eBPF 程序需要在受控的沙箱环境中运行，不能直接作为内核函数被调用

**简化理解**：

```
注册时：
用户空间的 eBPF 程序
        ↓ 编译成字节码
内核创建 "包装 probe" = __bpf_trace_run
        ↓
__bpf_trace_run 注册到 tracepoint

触发时：
tracepoint 触发
        ↓
调用 __bpf_trace_run（这是注册到 tracepoint 的 probe）
        ↓
__bpf_trace_run 内部调用 bpf_prog_run()
        ↓
执行真正的 eBPF 程序
```

---

## 9. 完整对比

### 9.1 使用方式对比

| 方式 | 需要写代码？ | 需要加载内核模块？ | 谁写 probe |
|------|-------------|------------------|-----------|
| ftrace | 否 | 否 | 内核写好了 |
| perf | 否 | 否 | 内核写好了 |
| tracepoint + 内核模块 | 是 | 是 | 你自己 |
| eBPF | 是 | 否 | 你自己 |

### 9.2 eBPF 监控 sched_switch

```c
// 写 eBPF 程序
SEC("tracepoint/sched/sched_switch")
int sched_switch(void *ctx)
{
    // 在内核执行你的逻辑
    return 0;
}
```

编译加载后，内核就会在 `sched_switch` 触发时调用你的函数。

---

## 10. 总结

### 10.1 tracepoint 核心设计

1. **一个函数 + 一个标志位**: 通过 `static_branch_unlikely` 实现零开销的条件判断
2. **动态注册**: probe 可以随时挂载和卸载
3. **文件系统接口**: 通过 sysfs/debugfs 提供用户空间控制能力

### 10.2 eBPF 核心设计

1. **沙箱**: 程序在受限环境中运行，不能随意访问内存
2. **验证器**: 内核检查程序是否安全
3. **JIT**: 编译成机器码，执行高效
4. **Map**: 用户空间和内核空间共享数据的方式

### 10.3 完整链路

```
用户空间工具 (ftrace/perf/eBPF)
    ↓ write() / bpf() 系统调用
sysfs/debugfs 文件节点 或 bpf() syscall
    ↓
tracepoint_probe_register()  ← eBPF/ftrace/perf 都走这里
    ↓
static_branch_enable()  启用 key
    ↓
调度器触发 → trace_sched_switch()
    ↓
static_branch_unlikely(key) 返回 true
    ↓
调用所有已注册的 probe
```

---

## 参考文件

| 文件 | 说明 |
|------|------|
| `include/trace/events/sched.h` | tracepoint 定义 |
| `include/linux/tracepoint.h` | tracepoint 宏定义 |
| `include/linux/tracepoint-defs.h` | 数据结构定义 |
| `kernel/trace/tracepoint.c` | 注册实现 |
| `kernel/sched/core.c` | 调度器触发点 |
| `kernel/trace/trace_events.c` | ftrace 事件处理 |
| `kernel/bpf/syscall.c` | eBPF 系统调用 |
| `kernel/trace/bpf_trace.c` | eBPF tracepoint 挂载 |
