---
show: step
version: 1.0
enable_checker: true
---

# tinyCoroLab4d实验解析

## tinyCoroLab4d验解析

> ⚠️tinyCoroLab的实验强烈推荐实验者独自完成而非直接翻阅实验解析，否则这与读完题直接翻看参考答案无太大区别，实验解析仅供实验者参考。

本节将会以tinyCoroLab的官方实现[tinyCoro](https://github.com/sakurs2/tinyCoro)为例，为大家分析并完成lab4d，请实验者预先下载tinyCoro的代码到本地。

```shell
git clone https://github.com/sakurs2/tinyCoro
```

并打开[include/coro/comp/mutex.hpp](https://github.com/sakurs2/tinyCoroLab/blob/master/include/coro/comp/mutex.hpp)和[src/comp/mutex.cpp](https://github.com/sakurs2/tinyCoroLab/blob/master/src/comp/mutex.cpp)并大致浏览代码结构。

## 📖lab4c任务参考实现

### 🧑‍💻Task #1 - 实现mutex

mutex的实现难点在于每次`unlock`只会恢复最多一个协程的运行，对于频繁`lock`和`unlock`的场景必须保证协程调度的正确性。

为了讲解方便，我先给出mutex实现的函数声明形式并给出注释讲解，这样更直观。

```cpp
class mutex
{
public:
    struct mutex_awaiter
    {
        mutex_awaiter(context& ctx, mutex& mtx) noexcept : m_ctx(ctx), m_mtx(mtx) {}

        constexpr auto await_ready() noexcept -> bool { return false; }

        auto await_suspend(std::coroutine_handle<> handle) noexcept -> bool;

        auto await_resume() noexcept -> void;

        // awaiter将自身注册到mutex的awaiter链表中，注册成功即陷入suspend状态
        auto register_lock() noexcept -> bool;

        // 对于恢复协程过程的封装，声明为虚函数是因为后续需要
        virtual auto resume() noexcept -> void;

        context&                m_ctx; // 绑定的context
        mutex&                  m_mtx; // 绑定的mutex
        mutex_awaiter*          m_next{nullptr}; // mutex协程链表的next指针
        std::coroutine_handle<> m_await_coro{nullptr}; // 待resume的协程
    };

    // 用于mutex.lock_guard的返回值，其逻辑与mutex_awaiter相同因此继承了mutex_awaiter，
    // 但重写了await_resume用于返回lock_guard
    struct mutex_guard_awaiter : public mutex_awaiter
    {
        using guard_type = detail::lock_guard<mutex>;
        using mutex_awaiter::mutex_awaiter;

        auto await_resume() noexcept -> guard_type
        {
            mutex_awaiter::await_resume(); // 保留原有的逻辑
            return guard_type(m_mtx);
        }
    };

public:
    mutex() noexcept : m_state(nolocked), m_resume_list_head(nullptr) {}
    ~mutex() noexcept { assert(m_state.load(std::memory_order_acquire) == mutex::nolocked); }

    auto try_lock() noexcept -> bool;

    auto lock() noexcept -> mutex_awaiter;

    auto unlock() noexcept -> void;

    auto lock_guard() noexcept -> mutex_guard_awaiter;

private:
    friend mutex_awaiter;

    // 常量1表示当前锁处于未加锁状态，常亮0表示锁处于加锁但没有协程在等待获取锁
    // 注意0和1不是随便设置的，原因稍后讲解
    inline static awaiter_ptr nolocked          = reinterpret_cast<awaiter_ptr>(1);
    inline static awaiter_ptr locked_no_waiting = 0; // nullptr
    std::atomic<awaiter_ptr>  m_state; // 与event类似，同样也是挂载suspend awaiter的链表头
    awaiter_ptr               m_resume_list_head; // 用于存储待恢复的suspend awaiter，作用稍后讲解
};
```

然后我们对重点函数的实现进行分析，首先是`mutex_awaiter`中关于协程调度的函数：

```cpp
auto mutex::mutex_awaiter::await_resume() noexcept -> void
{
    m_ctx.unregister_wait(); // 降低引用计数
}

auto mutex::mutex_awaiter::await_suspend(std::coroutine_handle<> handle) noexcept -> bool
{
    m_await_coro = handle; // 存储协程句柄
    m_ctx.register_wait(); // 增加引用计数 

    // 挂载自身到mutex的awaiter链表，挂载成功与否会决定协程
    // 是否陷入suspend状态
    return register_lock(); 
}
```

然后是`mutex_awaiter`挂载awaiter的逻辑：

```cpp
auto mutex::mutex_awaiter::register_lock() noexcept -> bool
{
    // 下列代码仍然使用了循环+cas的方法保证操作的原子性
    while (true)
    {
        auto state = m_mtx.m_state.load(std::memory_order_acquire);
        m_next     = nullptr;
        if (state == mutex::nolocked) // 如果未加锁
        {
            if (m_mtx.m_state.compare_exchange_weak(
                    state, mutex::locked_no_waiting, std::memory_order_acq_rel, std::memory_order_relaxed))
            {
                // 如果成功将锁置于locked_no_waiting状态，那么当前协程恢复运行
                return false;
            }
        }
        else
        {
            // 此时mutex已经被别的协程加锁，
            // state要么等于指向awaiter的指针，要么是locked_no_waiting
            // 如果state是locked_no_waiting，那么经过下述的操作当前的
            // awaiter的next指针则被置为locked_no_waiting，而locked_no_waiting
            // 值为0，即nullptr，这样正好方便了链表遍历

            // 将自身的next指向state并将自身置为state，
            // 这个操作lab4前几个实验也都有用到
            m_next = reinterpret_cast<mutex_awaiter*>(state);
            if (m_mtx.m_state.compare_exchange_weak(
                    state, reinterpret_cast<awaiter_ptr>(this), std::memory_order_acq_rel, std::memory_order_relaxed))
            {
                return true;
            }
        }
    }
}
```

然后是`mutex_awaiter`用于恢复协程执行的逻辑：

```cpp
auto mutex::mutex_awaiter::resume() noexcept -> void
{
    m_ctx.submit_task(m_await_coro); // 将协程句柄到其绑定的context的任务队列中
}
```

对于mutex的lock部分的接口其实现如下：

```cpp
auto mutex::try_lock() noexcept -> bool
{
    auto target = nolocked;
    return m_state.compare_exchange_strong(target, locked_no_waiting, std::memory_order_acq_rel, memory_order_relaxed);
}
auto mutex::lock() noexcept -> mutex_awaiter
{
    return mutex_awaiter(local_context(), *this);
}
auto mutex::lock_guard() noexcept -> mutex_guard_awaiter
{
    return mutex_guard_awaiter(local_context(), *this);
}
```

mutex的`unlock`实现相较复杂：

```cpp
auto mutex::unlock() noexcept -> void
{
    assert(m_state.load(std::memory_order_acquire) != nolocked && "unlock the mutex with unlock state");

    auto to_resume = reinterpret_cast<mutex_awaiter*>(m_resume_list_head);
    // 如果m_resume_list_head没空的话那么直接从m_resume_list_head取待恢复的协程就好了
    // 这样可以减少对原子变量的操作
    if (to_resume == nullptr)
    {
        // 此时m_resume_list_head为空，那么必须对m_state操作了
        auto target = locked_no_waiting;
        if (m_state.compare_exchange_strong(target, nolocked, std::memory_order_acq_rel, std::memory_order_relaxed))
        {
            // 此时没有任何suspend awaiter，那么直接返回
            return;
        }
        // 此时一定存在suspend awaiter

        // 将m_state上挂载的suspend awaiter全部取出来
        auto head = m_state.exchange(locked_no_waiting, std::memory_order_acq_rel);
        assert(head != nolocked && head != locked_no_waiting);

        // 下面是一个翻转链表操作，将去除的suspend awaiter链表翻转后
        // 挂载到m_resume_list_head上
        auto awaiter = reinterpret_cast<mutex_awaiter*>(head);
        do
        {
            auto temp       = awaiter->m_next;
            awaiter->m_next = to_resume;
            to_resume       = awaiter;
            awaiter         = temp;
        } while (awaiter != nullptr);
    }

    assert(to_resume != nullptr && "unexpected to_resume value: nullptr");
    // 取出 m_resume_list_head的链表头
    m_resume_list_head = to_resume->m_next;
    to_resume->resume();
}
```

注意上述`unlock`中对挂载awaiter的链表进行翻转操作，因此mutex的`unlock`操作遵从FIFO调度逻辑。

综合来看，对mutex的实现主要考察实验者对原子变量的使用。

## 实验总结

- **通过实现mutex加深了对原子变量的理解，合理的使用原子变量来构建无锁并发数据结构可以大大提高多线程编程的效率**
