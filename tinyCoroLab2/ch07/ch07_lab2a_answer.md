---
show: step
version: 1.0
enable_checker: true
---

# tinyCoroLab2a实验解析

## tinyCoroLab2a实验解析

> ⚠️tinyCoroLab的实验强烈推荐实验者独自完成而非直接翻阅实验解析，否则这与读完题直接翻看参考答案无太大区别，实验解析仅供实验者参考。

本节将会以tinyCoroLab的官方实现[tinyCoro](https://github.com/sakurs2/tinyCoro)为例，为大家分析并完成lab2a，请实验者预先下载tinyCoro的代码到本地。

```shell
git clone https://github.com/sakurs2/tinyCoro
```

并打开[include/coro/engine.hpp](https://github.com/sakurs2/tinyCoroLab/blob/master/include/coro/engine.hpp)和[src/engine.cpp](https://github.com/sakurs2/tinyCoroLab/blob/master/src/engine.cpp)并大致浏览代码结构。

## 📖lab2a任务参考实现

### 🧑‍💻Task #1 - 完善engine初始化以及异步I/O支持

对于异步IO支持部分，tinyCoro为engine添加了多个成员变量用来记录当前待提交的IO数以及正在运行且未完成的IO数：

```cpp
class engine{
  atomic<size_t> m_num_io_wait_submit{0};
  atomic<size_t> m_num_io_running{0};
};
```

由于IO状态的改变会涉及多个线程，因此均使用原子变量，对于`add_io_submit`实现如下：

```cpp
auto add_io_submit() noexcept -> void
{
    m_num_io_wait_submit.fetch_add(1, std::memory_order_release);
    wake_up(io_flag);
}
```

关于wake_up稍后解释，对于engine的初始化以及析构化，核心逻辑主要是对成员变量的处理及析构：

```cpp
auto engine::init() noexcept -> void
{
    linfo.egn            = this;
    m_num_io_wait_submit = 0;
    m_num_io_running     = 0;
    m_upxy.init(config::kEntryLength);
}
auto engine::deinit() noexcept -> void
{
    m_upxy.deinit();
    m_num_io_wait_submit = 0;
    m_num_io_running     = 0;
    mpmc_queue<coroutine_handle<>> task_queue;
    m_task_queue.swap(task_queue);
}
```

`get_free_urs`用于获取空闲的sqe，直接使用uring_proxy的方法即可：

```cpp
auto engine::get_free_urs() noexcept -> ursptr { return m_upxy.get_free_sqe(); }
```

#### 🧑‍💻Task #2 - 完善engine任务执行能力

有了成员变量来记录IO执行状态，那么`empty_io`的实现就很简单了：

```cpp
auto engine::empty_io() noexcept -> bool
{
    return m_num_io_wait_submit.load(std::memory_order_acquire) == 0 &&
            m_num_io_running.load(std::memory_order_acquire) == 0;
}
```

engine使用第三方库AtomicQueue存储协程句柄，因此关于非IO部分的任务处理只需要调用AtomicQueue提供的方法即可：

```cpp
auto engine::submit_task(coroutine_handle<> handle) noexcept -> void
{
    assert(handle != nullptr && "engine get nullptr task handle");
    m_task_queue.push(handle);
    wake_up();
}
auto engine::ready() noexcept -> bool { return !m_task_queue.was_empty(); }
auto engine::num_task_schedule() noexcept -> size_t { return m_task_queue.was_size(); }
auto engine::schedule() noexcept -> coroutine_handle<>
{
    auto coro = m_task_queue.pop();
    assert(bool(coro));
    return coro;
}
```

然后是IO的提交和执行部分，engine额外添加了一个成员函数用来提交IO：

```cpp
auto engine::do_io_submit() noexcept -> void
{
    int num_task_wait = m_num_io_wait_submit.load(std::memory_order_acquire);
    if (num_task_wait > 0)
    {
        int num = m_upxy.submit();
        num_task_wait -= num;
        assert(num_task_wait == 0);
        m_num_io_running.fetch_add(num, std::memory_order_acq_rel); // must set before m_num_io_wait_submit
        m_num_io_wait_submit.fetch_sub(num, std::memory_order_acq_rel);
    }
}
```

`do_io_submit`的逻辑很简单，如果有待提交的任务则调用uring_proxy的`submit`即可，然后对`m_num_io_running`和`m_num_io_wait_submit`进行更改。

对于核心的poll_submit，主要分为提交IO、等待IO执行、取出IO和处理已完成IO四个步骤，代码如下：

```cpp
auto engine::poll_submit() noexcept -> void
{
    do_io_submit(); // 提交IO

    auto cnt = m_upxy.wait_eventfd(); // 等待IO执行
    if (!wake_by_cqe(cnt))
    {
        return;
    }

    // 取出IO
    auto num = m_upxy.peek_batch_cqe(m_urc.data(), m_num_io_running.load(std::memory_order_acquire));

    if (num != 0)
    {
        // 处理IO
        for (int i = 0; i < num; i++)
        {
            handle_cqe_entry(m_urc[i]);
        }
        m_upxy.cq_advance(num);
        m_num_io_running.fetch_sub(num, std::memory_order_acq_rel);
    }
}
```

上述代码存在一个问题，实验者应该不难看出，**那就是工作线程在无任何任务的情况下利用阻塞在eventfd读操作上来让出执行权防止cpu空转。** 如果io_uring产生了cqe那么会向eventfd写值，此时工作线程被唤醒，继续执行任务，这没什么问题。

但是在长期运行工作模式下工作线程阻塞在了读eventfd，也没有IO任务完成，但scheduler向其派发了一个任务，那么问题来了，如何通知工作线程？在eventfd的帮助下这个问题的解决方案很简单，engine添加了一个函数`wake_up`专门用来唤醒工作线程：

```cpp
auto engine::wake_up(uint64_t val) noexcept -> void
{
    m_upxy.write_eventfd(val);
}
```

本质上是向eventfd写一个值，那么工作线程读取eventfd便可以立刻返回，那什么情况下需要调用wake_up呢？第一种当然是利用`submit_task`提交任务时，第二种是准备提交io时，这个需要特殊解释一下。

tinyCoro中的io_uring并没有开启polling模式，每次调用submit都会涉及到系统调用，因此为了优化此部分，仅在m_num_io_wait_submit大于0时才会提交，因此如果在工作线程阻塞状态下发起了IO，但此时工作线程并不会提交该IO，那么该IO便永远处于待提交状态，这也是为何`add_io_submit`会调用`wake_up`的原因。

综上工作线程被唤醒有三种情况并且可以同时发生：

- **case1：** 有新任务提交
- **case2：** 有新IO提交
- **case3：** 有IO已完成

为了区分这三种情况，engine定义了如下标志位：

```cpp
static constexpr uint64_t task_mask = (0xFFFFF00000000000);
static constexpr uint64_t io_mask   = (0x00000FFFFF000000);
static constexpr uint64_t cqe_mask  = (0x0000000000FFFFFF);

static constexpr uint64_t task_flag = (((uint64_t)1) << 44);
static constexpr uint64_t io_flag   = (((uint64_t)1) << 24);
```

即将64位整数的高20位分给case1，中间20位分给case2，其余24位分给case3，然后使用下述宏对从eventfd读取的值判断属于哪种case：

```cpp
#define wake_by_task(val) (((val) & engine::task_mask) > 0)
#define wake_by_io(val)   (((val) & engine::io_mask) > 0)
#define wake_by_cqe(val)  (((val) & engine::cqe_mask) > 0)
```

因此`poll_submit`包含下列判断语句来避免后续对uring_proxy的无效调用产生的开销：

```cpp
if (!wake_by_cqe(cnt))
{
    return;
}
```

> **💡如果频繁提交task那么会通过wake_up频繁向eventfd写值，岂不是开销很高？**
> 开销确实有，影响大小暂未评估，后续会作为优化点。

综上，engine的实现难点主要在于如何正确且高效的的避免工作线程一直陷入阻塞状态。

## 实验总结

- **本节通过完成engine学习了如何利用eventfd搭建一种高效的事件通知机制**
