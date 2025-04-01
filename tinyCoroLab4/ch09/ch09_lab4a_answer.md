---
show: step
version: 1.0
enable_checker: true
---

# tinyCoroLab4a实验解析

## tinyCoroLab4a实验解析

> ⚠️tinyCoroLab的实验强烈推荐实验者独自完成而非直接翻阅实验解析，否则这与读完题直接翻看参考答案无太大区别，实验解析仅供实验者参考。

本节将会以tinyCoroLab的官方实现[tinyCoro](https://github.com/sakurs2/tinyCoro)为例，为大家分析并完成lab4a，请实验者预先下载tinyCoro的代码到本地。

```shell
git clone https://github.com/sakurs2/tinyCoro
```

并打开[include/coro/comp/event.hpp](https://github.com/sakurs2/tinyCoroLab/blob/master/include/coro/comp/event.hpp)和[src/comp/event.cpp](https://github.com/sakurs2/tinyCoroLab/blob/master/src/comp/event.cpp)并大致浏览代码结构。

## 📖lab4a任务参考实现

### 🧑‍💻Task #1 - 实现event

因为event针对模板参数是否为空具有不同的行为，因此分离出event_base来实现共性行为（调度逻辑），其具有唯一一个成员变量用来挂载suspend awaiter：

```cpp
std::atomic<awaiter_ptr> event_base::m_state{nullptr};
```

**当`m_state`等于event的this指针时表示event此时已被set。**

下面给出event_base的定义并标明注释：

```cpp
class event_base
{
public:
    struct awaiter_base
    {
        awaiter_base(context& ctx, event_base& e) noexcept : m_ctx(ctx), m_ev(e) {}

        inline auto next() noexcept -> awaiter_base* { return m_next; }

        auto await_ready() noexcept -> bool;

        auto await_suspend(std::coroutine_handle<> handle) noexcept -> bool;

        auto await_resume() noexcept -> void;

        context&                m_ctx; // 绑定的context
        event_base&             m_ev; // 绑定的event
        awaiter_base*           m_next{nullptr}; // 链表的next指针
        std::coroutine_handle<> m_await_coro{nullptr}; // 待resume的协程句柄
    };

    event_base(bool initial_set = false) noexcept : m_state((initial_set) ? this : nullptr) {}
    ~event_base() noexcept = default;

    event_base(const event_base&)            = delete;
    event_base(event_base&&)                 = delete;
    event_base& operator=(const event_base&) = delete;
    event_base& operator=(event_base&&)      = delete;

    // 判断event是否被set，根据m_state是否等于this
    inline auto is_set() const noexcept -> bool { return m_state.load(std::memory_order_acquire) == this; }

    auto set_state() noexcept -> void; // 设置event，唤醒所有suspend awaiter

    auto resume_all_awaiter(awaiter_ptr waiter) noexcept -> void; // 唤醒所有suspend awaiter

    auto register_awaiter(awaiter_base* waiter) noexcept -> bool; // 挂载suspend awaiter

private:
    std::atomic<awaiter_ptr> m_state{nullptr};
};
```

首先针对awaiter_base结构体，其核心成员变量主要是用于帮助该awaiter在陷入suspend状态后正确恢复，另外next指针用于指向下一个suspend awaiter，因为event挂载awaiter的形式是链表，event_base中的`m_state`表示链表头。

下面分析awaiter_base成员函数的具体实现并附加注释：

```cpp
auto event_base::awaiter_base::await_ready() noexcept -> bool
{
    m_ctx.register_wait(); // 增加引用计数
    return m_ev.is_set(); // 如果event是否被设置决定协程是否陷入suspend
}

auto event_base::awaiter_base::await_suspend(std::coroutine_handle<> handle) noexcept -> bool
{
    m_await_coro = handle; // 保存协程句柄

    // 将协程awaiter挂载到event中，这个过程期间有可能event被set，因此返回bool表示是否
    // 挂载成功，挂载失败那么协程恢复运行
    return m_ev.register_awaiter(this); 
}

auto event_base::awaiter_base::await_resume() noexcept -> void
{
    m_ctx.unregister_wait(); // 降低引用计数
}
```

对于event_base的`set_state`实现如下：

```cpp
auto event_base::set_state() noexcept -> void
{
    auto flag = m_state.exchange(this, std::memory_order_acq_rel);
    if (flag != this) // event之前未被set
    {
        auto waiter = static_cast<awaiter_base*>(flag);
        resume_all_awaiter(waiter); // 取得列表头并恢复所有suspend awaiter
    }
}
```

对于event_base的`resume_all_awaiter`实现如下：

```cpp
auto event_base::resume_all_awaiter(awaiter_ptr waiter) noexcept -> void
{
    // 以循环的形式遍历链表
    while (waiter != nullptr)
    {
        auto cur = static_cast<awaiter_base*>(waiter);

        // 将该awaiter绑定的协程投放到其绑定的context的任务队列里
        cur->m_ctx.submit_task(cur->m_await_coro);
        waiter = cur->m_next;
    }
}
```

> **💡为何event挂载suspend awaiter采用链表形式？**
> 挂载的awaiter数量不确定，如果event使用容器的话可能产生动态内存分配开销，不如直接在awaiter中多添加一个next指针，这样event只需要存储一个链表头就可以了。

对于event_base的`register_awaiter`实现如下：

```cpp
auto event_base::register_awaiter(awaiter_base* waiter) noexcept -> bool
{
    const auto  set_state = this;
    awaiter_ptr old_value = nullptr;
    // 利用cas操作确保挂载操作的原子性
    do
    {
        old_value = m_state.load(std::memory_order_acquire);
        if (old_value == this)
        {
            waiter->m_next = nullptr;
            return false;
        }
        waiter->m_next = static_cast<awaiter_base*>(old_value);
    } while (!m_state.compare_exchange_weak(old_value, waiter, std::memory_order_acquire));

    return true;
}
```

上述代码是典型的原子变量cas的使用场景，当前awaiter将next指针指向event_base的m_state并将自身置为新的m_state，相当于后挂载的awaiter会作为最新的链表头，因为整个操作非原子，所以使用`compare_exchange_weak`判断是否在这个过程中有其他线程修改了状态，因为是do while循环，所以使用`compare_exchange_weak`而不是`compare_exchange_strong`。

> **💡后挂载的awaiter会作为最新的链表头，恢复协程的时候如果按链表遍历顺序恢复那岂不LIFO调度方式？**
> 是的，如果想实现FIFO那么就在取出链表后对其进行反转再按链表遍历顺序恢复。

综上，event_base实现的协程与恢复机制并不复杂，只是需要注意对原子变量`m_state`的正确操作。

模板参数为空的event只需要继承event_base后调用其接口就可以了。

模板参数非空的event可以选择继承event_base和container，在存储与取出数据时调用container相关的接口即可，需要注意的是此时`wait`返回的awaiter需要在awaiter_base的基础上修改`await_resume`函数并将container存储的值返回，代码如下：

```cpp
auto await_resume() noexcept -> decltype(auto)
{
    detail::event_base::awaiter_base::await_resume(); // 保留降低引用计数的代码
    return static_cast<event&>(m_ev).result();
}
```

## 实验总结

- **通过实现event的awaiter挂载机制了解了链表的一种高效应用场景**
- **学会了对多线程编程下对原子变量的正确操作**
