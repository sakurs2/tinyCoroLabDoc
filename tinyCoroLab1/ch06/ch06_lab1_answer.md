---
show: step
version: 1.0
enable_checker: true
---

# tinyCoroLab1实验解析

## tinyCoroLab1实验解析

> ⚠️tinyCoroLab的实验强烈推荐实验者独自完成而非直接翻阅实验解析，否则这与读完题直接翻看参考答案无太大区别，实验解析仅供实验者参考。

本节将会以tinyCoroLab的官方实现[tinyCoro](https://github.com/sakurs2/tinyCoro)为例，为大家分析并完成lab1，请实验者预先下载tinyCoro的代码到本地。

```shell
git clone https://github.com/sakurs2/tinyCoro
```

并打开[include/coro/task.hpp](https://github.com/sakurs2/tinyCoroLab/blob/master/include/coro/task.hpp)并大致浏览代码结构。

## 📖lab1任务参考实现

### 🧑‍💻Task #1 - 实现task执行结束后的正确调度逻辑

要想实现任务目标中的调度逻辑，核心便是建立协程调用之间的互联关系，即让被调用协程可以感知到调用协程，这样在被调用协程执行完成后才可以正确转移执行权到用协程，这样就形成了正确的嵌套调用关系，那么如何建立协程调用之间的互联关系呢？

tinyCoro为promise_base添加了`m_continuation`成员用于表示协程的父级调用协程的句柄以及用于设置该成员的函数`continuation`，定义如下：

```cpp
struct promise_base
{
    std::coroutine_handle<> m_continuation{nullptr};
    auto continuation(std::coroutine_handle<> continuation) noexcept -> void { m_continuation = continuation; }
};
```

那么什么时候调用`continuation`函数呢？在task内部存在结构体`awaitable_base`作为task被co_await调用时返回的awaiter，这通过重载co_await运算符来实现，**实验者应该不难理解`awaitable_base`的`await_suspend`函数的参数是调用协程的句柄，因为`awaitable_base`是在被调用协程对co_await重载后的函数内被返回，所以本身可以持有被调用协程的句柄，那么此时获取被调用协程的promise并调用函数`continuation`，参数即`awaitable_base`的`await_suspend`的参数。**

此时已经为调用和被调用协程建立互联关系了，下一步就是如何使用互联关系。tinyCoro添加了结构体`final_awaitable`作为`final_suspend`的返回值，定义如下：

```cpp
struct final_awaitable
{
    constexpr auto await_ready() const noexcept -> bool { return false; }

    template<typename promise_type>
    auto await_suspend(std::coroutine_handle<promise_type> coroutine) noexcept -> std::coroutine_handle<>
    {
        // If there is a continuation call it, otherwise this is the end of the line.
        auto& promise = coroutine.promise();
        return promise.m_continuation != nullptr ? promise.m_continuation : std::noop_coroutine();
    }

    constexpr auto await_resume() noexcept -> void {}
};
```

`final_awaitable`通过在`await_suspend`中获取promise的`m_continuation`来决定执行权的转移，如果`m_continuation`非空即存在父调用协程，然后转移执行权。

> **💡当`await_suspend`返回`std::noop_coroutine()`时执行权如何转移呢？**
> 在预备知识章节已经讲过了哦！

### Task #2 - 为task添加detach状态

状态是需要被存储的，所以tinyCoro为promise_base添加了一个状态成员以及与状态相关的成员函数：

```cpp
enum class coro_state : uint8_t
{
    normal,
    detach,
    none
};
struct promise_base
{
    inline auto set_state(coro_state state) -> void { m_state = state; }
    inline auto get_state() -> coro_state { return m_state; }
    inline auto is_detach() -> bool { return m_state == coro_state::detach; }

    coro_state  m_state{coro_state::normal};
};
```

对于task的`detach`函数，对其关联的promise设置状态就可以了，并且移除自身持有的协程句柄，代码如下：

```cpp
auto task::detach() -> void
{
    assert(m_coroutine != nullptr && "detach func expected no-nullptr coroutine_handler");
    auto& promise = m_coroutine.promise();
    promise.set_state(detail::coro_state::detach);
    m_coroutine = nullptr;
}
```

而负责协程资源清理的`clean`函数，只要判断协程是否为detach状态，是则释放资源：

```cpp
inline auto clean(std::coroutine_handle<> handle) noexcept -> void
{
    auto  specific_handle = coroutine_handle::from_address(handle.address());
    auto& promise         = specific_handle.promise();
    switch (promise.get_state())
    {
        case detail::coro_state::detach:
            handle.destroy();
            break;
        default:
            break;
    }
}
```

## 实验总结

- **通过完成协程的嵌套调用对协程执行权转移以及如何利用awaiter转移有了更深的理解**
- **通过完成task的detach状态学习到了如何正确处理协程持有的资源避免产生内存安全问题**
