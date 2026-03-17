+++
date = '2026-03-17T13:11:32+08:00'
draft = false
title = 'DPDK定时器（1）：基于跳表的设计与初始化'
+++

本系列选择 DPDK 25.07 版本代码，对 `rte_timer` 定时器子系统进行分析。

本篇文章主要聚焦于以下几个问题：

- **DPDK 定时器的整体设计思路**：为什么内部实现不是时间轮，而是基于跳表（skip list）的有序链表？
- **定时器子系统的初始化流程**：从全局子系统到单个定时器对象，初始化时到底做了什么？

后续文章会分别展开 `rte_timer_reset` 等核心操作的分析。

---

## 1. 基础概念：rte_timer 做的是什么？

`rte_timer` 提供的是一个**高性能、多核安全**的定时器库，用来做下面这些典型事情：

- 连接超时检测
- 周期性统计、写日志
- 资源清理（如 session、缓存项回收）
- 各种“延时执行某个回调函数”的场景

从用户视角看，它就是：

- 定义一个 `struct rte_timer`
- 通过 `rte_timer_init` 初始化
- 通过 `rte_timer_reset` 设定触发时间 / 周期、回调函数、在哪个 lcore 上执行
- 由某个核心周期性调用 `rte_timer_manage`，到期就执行回调

---

## 2. 时间轮

直觉上，很多人会先想到**时间轮**（timer wheel）。它的模型非常直观：

- 将时间轴均分成若干槽位（slots），每个槽存放一定时间范围内要触发的定时器；
- 当前“时间指针”走到某个槽位，就取出该槽中的所有定时器并执行回调；
- 为了让时间可以不断前进，时间轴通常做成环形缓冲区（wheel），指针绕圈前进，因此叫“时间轮”。

**优点：**

- 插入复杂度通常是 \(O(1)\)；
- 实现思路简单，直观。

**缺点：**

- 必须有固定的时间粒度（刻度），精度和可表示的时间范围都受限；
- 如果想“既支持很长的超时，又要保证较高精度”，往往要做多级时间轮，设计复杂度和实现成本迅速上升。

对于一个**通用高性能网络编程框架**来说，这些约束有点重：既要能支持各种业务的时间范围，又不想把自己绑死在某种固定的时间刻度设计上。

---

## 3. 跳表

### 3.1 基本思想

跳表的出发点是：把一个普通的有序单链表，**“堆叠”出多层稀疏索引**。

- 第 0 层：完整有序链表；
- 第 1 层：从第 0 层抽样（大约 1/2 的节点）；
- 第 2 层：再从第 1 层抽样（大约 1/2）；
- ……

查找时的过程类似“从上往下逐层缩小范围”：

- 先在最高层往前走，一跳可以跨过很多节点；
- 在某层发现“再往前走会超过目标值”时，就下降到下一层；
- 最后在第 0 层精确定位，复杂度可以做到平均 \(O(\log n)\)。

### 3.2 随机化抽样

实际实现中，不会硬性规定“每隔 2 个节点抽一个”——那样插入会很难维护。跳表采用**随机化抽样**：

- 每插入一个新节点，用“掷硬币”的方式决定它是否出现在上一层；
- 每一层大约有下一层 1/2 或 1/4 的节点数量（取决于实现的概率设定）；
- 这样可以在统计意义上保持平衡，而不需要像平衡树那样做复杂旋转。

### 3.3 跳表的典型操作

- **查找**：从最高层开始，从左往右走，遇到再走会“超过目标”的节点就下到下一层，直到第 0 层；
- **插入**：先按查找流程找到每层的前驱节点，再随机决定新节点的层数，把它插入各层；
- **删除**：同样先找到各层前驱节点，再在对应层的链中把它摘掉。如果某一高层空了，可以降低跳表高度。

---

## 4. 为什么 DPDK 选择了跳表而不是时间轮？

时间轮在“常数时间插入”上确实很优秀，但 DPDK 作为一个**通用、高性能、可复用的基础设施**，还有更多考虑：

- **时间范围要足够大**：既要支持非常短的周期任务，也要能支撑比较长的超时；
- **精度不能被槽宽束缚**：业务可能要求较高精度，固定槽宽不太灵活；

而跳表的特点刚好对 DPDK 友好：

- 内部是一条按照 `expire` 升序的有序链表，不依赖离散时间槽；
- 可以支持很大的时间范围，也不需要预先定义粒度；
- 在节点数比较大的情况下，插入 / 删除 / 查找都可以保持不错的平均复杂度。

从“**通用、高性能、范围广**”这个角度看，跳表相比时间轮更适合 `rte_timer` 的角色。

---

## 5. 核心数据结构概览

### 5.1 单个定时器节点：rte_timer

定时器本身的结构体如下（省略了一些宏）：

```c
struct rte_timer {
    uint64_t expire;                       /**< Time when timer expire. */
    struct rte_timer *sl_next[MAX_SKIPLIST_DEPTH];
    volatile union rte_timer_status status;/**< Status of timer. */
    uint64_t period;                       /**< Period of timer (0 if not periodic). */
    rte_timer_cb_t f;                      /**< Callback function. */
    void *arg;                             /**< Argument to callback function. */
};
```

可以简单理解成：

- `expire`：定时器的“到期时间”，按这个字段在跳表中有序排列；
- `sl_next[]`：每一层的“下一跳”指针，实现多层索引；
- `status`：状态 + 所属 lcore，使用原子类型保证并发安全；
- `period`：周期性定时器的周期长度，如果为 0 表示单次定时器；
- `f` / `arg`：定时器到期后要执行的回调函数及其参数。

### 5.2 每核管理结构：priv_timer

每个 lcore 上都有一份自己的“定时器管理状态”：

```c
struct __rte_cache_aligned priv_timer {
    struct rte_timer pending_head;  /**< dummy timer instance to head up list */
    rte_spinlock_t list_lock;       /**< lock to protect list access */
    int updated;                    /**< indicate this core updated timer since last check */
    unsigned curr_skiplist_depth;   /**< current depth of the skiplist */
    unsigned prev_lcore;            /**< used for lcore round robin */
    struct rte_timer *running_tim;  /**< running timer on this lcore now */
#ifdef RTE_LIBRTE_TIMER_DEBUG
    struct rte_timer_debug_stats stats;
#endif
};

struct rte_timer_data {
    struct priv_timer priv_timer[RTE_MAX_LCORE];
    uint8_t internal_flags;
};
```

其中几个关键字段：

- `pending_head`：跳表的哨兵头节点，本身不存放真实定时器，主要是为了简化插入/删除逻辑；
- `list_lock`：保护该 lcore 上跳表的自旋锁；
- `updated`：该核上是否对定时器做过 `reset/stop` 等更新操作；
- `curr_skiplist_depth`：当前跳表的“高度”（层数），是一个左闭右开的概念——最高层本身没有真实节点，真正的数据从下一层开始；
- `prev_lcore`：用于 round-robin 分配定时器归属到各个 lcore；
- `running_tim`：当前正在这个 lcore 上执行的定时器指针。

`rte_timer_data` 则是**整个定时器子系统的顶层结构**，内部是一组 `priv_timer`，每个 lcore 对应一个。

---

## 6. 定时器子系统初始化：rte_timer_subsystem_init

在使用任何 `rte_timer` 相关 API 之前，必须先初始化定时器子系统：

```c
int rte_timer_subsystem_init(void)
{
    const struct rte_memzone *mz;
    struct rte_timer_data *data;
    int i, lcore_id;
    static const char *mz_name = "rte_timer_mz";
    const size_t data_arr_size =
            RTE_MAX_DATA_ELS * sizeof(*rte_timer_data_arr);
    const size_t mem_size = data_arr_size + sizeof(*rte_timer_mz_refcnt);
    bool do_full_init = true;

    rte_mcfg_timer_lock();
    if (rte_timer_subsystem_initialized) {
        rte_mcfg_timer_unlock();
        return -EALREADY;
    }

    mz = rte_memzone_lookup(mz_name);
    if (mz == NULL) {
        mz = rte_memzone_reserve_aligned(mz_name, mem_size,
                SOCKET_ID_ANY, 0, RTE_CACHE_LINE_SIZE);
        if (mz == NULL) {
            rte_mcfg_timer_unlock();
            return -ENOMEM;
        }
        do_full_init = true;
    } else
        do_full_init = false;

    rte_timer_data_mz = mz;
    rte_timer_data_arr = mz->addr;
    rte_timer_mz_refcnt =
        (void *)((char *)mz->addr + data_arr_size);

    if (do_full_init) {
        for (i = 0; i < RTE_MAX_DATA_ELS; i++) {
            data = &rte_timer_data_arr[i];

            for (lcore_id = 0; lcore_id < RTE_MAX_LCORE;
                 lcore_id++) {
                rte_spinlock_init(
                    &data->priv_timer[lcore_id].list_lock);
                data->priv_timer[lcore_id].prev_lcore =
                    lcore_id;
            }
        }
    }

    rte_timer_data_arr[default_data_id].internal_flags |= FL_ALLOCATED;
    (*rte_timer_mz_refcnt)++;
    rte_timer_subsystem_initialized = 1;
    rte_mcfg_timer_unlock();

    return 0;
}
```

可以拆成几个关键步骤来看：

### 6.1 加全局锁 + 防重复初始化

- `rte_mcfg_timer_lock()`：给“定时器配置相关的全局状态”加锁；
- 如果已经初始化过（`rte_timer_subsystem_initialized`），直接返回 `-EALREADY`，避免重复初始化。

### 6.2 在共享内存中分配管理结构

DPDK 用 `rte_memzone` 为定时器子系统分配一块共享内存：

- `rte_timer_data_arr`：是一个数组，元素类型为 `struct rte_timer_data`；
- 之所以是“数组”，是为了支持**多个独立的定时器系统实例**——比如你写了一个库，库本身用定时器，使用方的应用程序也想有自己的定时器系统，就可以用不同的 `timer_data_id` 互不干扰；
- 默认 API（例如普通的 `rte_timer_reset`）使用的是 `default_data_id` 指向的那一个。

这也是为什么代码里会有 `RTE_MAX_DATA_ELS` 这样一个“最多多少个定时器系统”的上限。

### 6.3 全量初始化：每个 lcore 的 priv_timer

如果是**第一次**分配这块 memzone（`do_full_init == true`），则需要：

- 遍历所有 `rte_timer_data_arr[i]`；
- 对每个 `rte_timer_data`，遍历所有 `lcore_id`：
  - 初始化 `list_lock`；
  - 把 `prev_lcore` 初始化为它自己的 `lcore_id`（为后面的 round-robin 做准备）。

否则，如果 memzone 之前已经存在，说明这些初始化工作早就做过，就不再重复。

### 6.4 设置标记 + 引用计数

最后，DPDK 会：

- 设置默认定时器系统的 `internal_flags |= FL_ALLOCATED`，表示这一格已经被使用；
- 增加 memzone 的引用计数 `(*rte_timer_mz_refcnt)++`；
- 设置 `rte_timer_subsystem_initialized = 1`；
- 解锁并返回。

到这里为止，“全局的定时器子系统”已经就绪，但单个定时器对象还没有初始化。

---

## 7. 初始化单个定时器：rte_timer_init

在子系统就绪之后，用户需要为每个定时器对象调用 `rte_timer_init`：

```c
#define RTE_TIMER_STOP    0 /**< State: timer is stopped. */
#define RTE_TIMER_PENDING 1 /**< State: timer is scheduled. */
#define RTE_TIMER_RUNNING 2 /**< State: timer function is running. */
#define RTE_TIMER_CONFIG  3 /**< State: timer is being configured. */
#define RTE_TIMER_NO_OWNER -2 /**< Timer has no owner. */

union rte_timer_status {
    struct {
        RTE_ATOMIC(uint16_t) state;  /**< Stop, pending, running, config. */
        RTE_ATOMIC(int16_t) owner;   /**< The lcore that owns the timer. */
    };
    RTE_ATOMIC(uint32_t) u32;        /**< To atomic-set status + owner. */
};

void
rte_timer_init(struct rte_timer *tim)
{
    union rte_timer_status status;

    status.state = RTE_TIMER_STOP;
    status.owner = RTE_TIMER_NO_OWNER;
    rte_atomic_store_explicit(&tim->status.u32,
                              status.u32,
                              rte_memory_order_relaxed);
}
```

主要做的事情只有两步：

1. 把 `state` 设置为 `RTE_TIMER_STOP`，表示“目前是停止状态”；
2. 把 `owner` 设置为 `RTE_TIMER_NO_OWNER`，表示“还没有任何 lcore 拥有它”。

这里有两个点值得注意：

- `status` 设计成 `union`，既可以按 `{ state, owner }` 看，也可以当作一个 `u32` 一次性原子更新，避免“先改 state 再改 owner”被编译器或 CPU 重排；
- 这里使用的内存序是 `rte_memory_order_relaxed`，原因在于初始化本身不参与跨线程同步语义——DPDK 的规范要求：你必须在资源初始化完成之后再并发使用，如果你在另一个线程中同时对同一个 `tim` 调用 `rte_timer_reset`，那是调用者的逻辑错误，而不是库要保证的事情。

因此在 init 阶段，`relaxed` 就足够了，既能保证单次写入的原子性，也不会给 CPU 增加不必要的栅栏开销。

---

## 8. 小结

这一篇我们完成了三件事：

- 从宏观上解释了 **为什么 DPDK 定时器采用跳表而不是时间轮**；
- 介绍了定时器子系统的**核心数据结构**：`rte_timer`、`priv_timer`、`rte_timer_data`；
- 读了 **子系统初始化** 和 **单个定时器初始化** 的关键流程。


下一篇将聚焦于对 `rte_timer_reset` 的研究。

> 免责声明：本文内容为我个人在阅读 DPDK 源码过程中的梳理与理解，难免存在疏漏或误解。如有不正确之处，欢迎批评指正。