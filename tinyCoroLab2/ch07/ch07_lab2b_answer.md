---
show: step
version: 1.0
enable_checker: true
---

# tinyCoroLab2b实验解析

## tinyCoroLab2b实验解析

> ⚠️tinyCoroLab的实验强烈推荐实验者独自完成而非直接翻阅实验解析，否则这与读完题直接翻看参考答案无太大区别，实验解析仅供实验者参考。

本节将会以tinyCoroLab的官方实现[tinyCoro](https://github.com/sakurs2/tinyCoro)为例，为大家分析并完成lab2b，请实验者预先下载tinyCoro的代码到本地。

```shell
git clone https://github.com/sakurs2/tinyCoro
```

并打开[include/coro/context.hpp](https://github.com/sakurs2/tinyCoroLab/blob/master/include/coro/context.hpp)和[src/context.cpp](https://github.com/sakurs2/tinyCoroLab/blob/master/src/context.cpp)并大致浏览代码结构。

## 📖lab2b任务参考实现

### 🧑‍💻Task #1 - 完善context初始化以及外部交互API

context的初始化与析构化比较简单：

```cpp
auto context::init() noexcept -> void
{
    linfo.ctx = this;
    m_engine.init();
}
auto context::deinit() noexcept -> void
{
    linfo.ctx = nullptr;
    m_engine.deinit();
}
```

对于context的提交任务只需要调用engine的接口就可以了：

```cpp
auto context::submit_task(std::coroutine_handle<> handle) noexcept -> void
{
    m_engine.submit_task(handle);
}
```

context的引用计数可能涉及到多线程操作，因此使用原子变量存储计数，`register_wait`和`unregister_wait`方法只需要对计数增减即可。

```cpp
atomic<size_t>      context::m_num_wait_task{0};
auto context::register_wait(int register_cnt = 1) noexcept -> void 
{
    m_num_wait_task.fetch_add(size_t(register_cnt), memory_order_acq_rel);
}
auto context::unregister_wait(int register_cnt = 1) noexcept -> void
{
    m_num_wait_task.fetch_sub(register_cnt, memory_order_acq_rel);
}
```

然后是通知工作线程停止的`notify_stop`，使用jthread提供的方法并注意调用`wake_up`，因为此时工作线程可能在阻塞态，代码如下：

```cpp
auto context::notify_stop() noexcept -> void
{
    m_job->request_stop();
    m_engine.wake_up();
}
```

### 🧑‍💻Task #2 - 完善context核心任务循环

首先为context添加一些辅助函数比如执行任务、执行IO任务以及状态判断：

```cpp
// 驱动engine从任务队列取出任务并执行
auto context::process_work() noexcept -> void
{
    auto num = m_engine.num_task_schedule();
    for (int i = 0; i < num; i++)
    {
        m_engine.exec_one_task();
    }
}
// 驱动engine执行IO任务
auto context::poll_work() noexcept -> void { m_engine.poll_submit(); }
// 判断是否没有IO任务以及引用计数是否为0
auto context::empty_wait_task() noexcept -> bool
{
    return m_num_wait_task.load(memory_order_acquire) == 0 && m_engine.empty_io();
}
```

对于任务循环函数`run`，其核心目标便是在一遍遍的循环中驱动engine完成全部任务且优雅的退出，tinyCoro给出的实现如下：

```cpp
auto context::run(stop_token token) noexcept -> void
{
    while (true)
    {
        process_work();
        if (token.stop_requested() && empty_wait_task())
        {
            if (!m_engine.ready())
            {
                break;
            }
            else // 此时任务队列还有任务，优先执行任务队列里的任务
            {
                continue;
            }
        }
        poll_work();
        if (token.stop_requested() && empty_wait_task() && !m_engine.ready())
        {
            break;
        }
    }
}
```

tinyCoro对`run`函数的实现并非标准实现，实验者只需要保证实现满足核心目标即可。

## 实验总结

- **通过完善context驱动engine执行任务使得tinyCoro正式拥有完整的对外提供服务的能力**
