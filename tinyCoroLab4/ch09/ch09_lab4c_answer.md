---
show: step
version: 1.0
enable_checker: true
---

# tinyCoroLab4c实验解析

## tinyCoroLab4c实验解析

> ⚠️tinyCoroLab的实验强烈推荐实验者独自完成而非直接翻阅实验解析，否则这与读完题直接翻看参考答案无太大区别，实验解析仅供实验者参考。

本节将会以tinyCoroLab的官方实现[tinyCoro](https://github.com/sakurs2/tinyCoro)为例，为大家分析并完成lab4c，请实验者预先下载tinyCoro的代码到本地。

```shell
git clone https://github.com/sakurs2/tinyCoro
```

并打开[include/coro/comp/wait_group.hpp](https://github.com/sakurs2/tinyCoroLab/blob/master/include/coro/comp/wait_group.hpp)和[src/comp/wait_group.cpp](https://github.com/sakurs2/tinyCoroLab/blob/master/src/comp/wait_group.cpp)并大致浏览代码结构。

## 📖lab4c任务参考实现

### 🧑‍💻Task #1 - 实现wait_group

wait_group相比latch仅多了一个可以添加计数的功能，但却不能使用event来简化实现了，因为event一旦被set后续的协程调用`wait`均不会陷入suspend状态，而wait_group在引用计数降至0后可以重新增加计数，相当于将event重置为unset状态，而tinyCoro的event不具备这个功能，因此需要自行实现逻辑，但实验者也可以将event实现为可以被unset这样wait_group又可以被event用来简化实现了。

首先看wait_group的声明：

```cpp
class wait_group
{
public:
    struct awaiter
    {
        awaiter(context& ctx, wait_group& wg) noexcept : m_ctx(ctx), m_wg(wg) {}

        constexpr auto await_ready() noexcept -> bool { return false; }

        auto await_suspend(std::coroutine_handle<> handle) noexcept -> bool;

        auto await_resume() noexcept -> void;

        auto resume() noexcept -> void;

        context&                m_ctx; // 绑定的context
        wait_group&             m_wg; // 绑定的wait_group
        awaiter*                m_next{nullptr}; // suspend awaiter链表的next指针
        std::coroutine_handle<> m_await_coro{nullptr}; // 待resume的协程句柄
    };

    explicit wait_group(int count = 0) noexcept : m_count(count) {}

    auto add(int count) noexcept -> void;

    auto done() noexcept -> void;

    auto wait() noexcept -> awaiter;

private:
    friend awaiter;
    std::atomic<int32_t>     m_count; // 保存计数
    std::atomic<awaiter_ptr> m_state; // 与event的m_state功能一致
};
```

下面分析awaiter的实现，对于`await_ready`其恒定返回false，则`await_suspend`一定会被调用，当然实验者自己对`await_ready`的实现也可以不返回恒定值。

对于`await_suspend`其实现如下：

```cpp
auto wait_group::awaiter::await_suspend(std::coroutine_handle<> handle) noexcept -> bool
{
    m_await_coro = handle; // 保存协程句柄
    m_ctx.register_wait(); // 增加引用计数
    while (true)
    {
        // 如果引用计数为0那么协程不用陷入suspend状态
        if (m_wg.m_count.load(std::memory_order_acquire) == 0)
        {
            return false;
        }

        // 下面的操作主要是当前awaiter将自身置为wait_group的链表头，
        // 然后将next指向原先的链表头
        auto head = m_wg.m_state.load(std::memory_order_acquire);
        m_next    = static_cast<awaiter*>(head);
        if (m_wg.m_state.compare_exchange_weak(
                head, static_cast<awaiter_ptr>(this), std::memory_order_acq_rel, std::memory_order_relaxed))
        {
            // 引用计数不为0且成功挂载至wait_group的awaiter链表
            return true;
        }
    }
}
```

上述代码需要注意的是awaiter对于wait_group的m_state即链表头的操作，由于整个系列操作是非原子的，因此用原子变量的cas来判断过程中是否有其他线程改变了状态，因为是循环操作，所以使用了`compare_exchange_weak`而不是`compare_exchange_strong`，这部分代码与lab4a中event的实现逻辑是相同的。

`await_resume`主要实现降低引用计数的逻辑：

```cpp
auto wait_group::awaiter::await_resume() noexcept -> void
{
    m_ctx.unregister_wait();
}
```

另外awaiter还提供了`resume`函数用来封装恢复协程执行的逻辑：

```cpp
auto wait_group::awaiter::resume() noexcept -> void
{
    m_ctx.submit_task(m_await_coro); // 将协程句柄提交到其绑定的context的任务队列
}
```

对于wait_group面向用户的接口，其实现如下：

```cpp
auto wait_group::add(int count) noexcept -> void
{
    m_count.fetch_add(count, std::memory_order_acq_rel); // 增加计数
}

auto wait_group::done() noexcept -> void
{
    if (m_count.fetch_sub(1, std::memory_order_acq_rel) == 1)
    {
        // 如果计数降至0，那么恢复所有的suspend awaiter
        auto head = static_cast<awaiter*>(m_state.exchange(nullptr, std::memory_order_acq_rel));
        while (head != nullptr)
        {
            head->resume();
            head = head->m_next;
        }
    }
}

auto wait_group::wait() noexcept -> awaiter
{
    return awaiter{local_context(), *this};
}
```

上述的代码实验者在完成lab4a和lab4b应当相当熟悉，因为逻辑是一样的。

## 实验

- **通过实现wait_group完成了拓展版的latch，不管功能差异多大，其本质实现是一样的**
