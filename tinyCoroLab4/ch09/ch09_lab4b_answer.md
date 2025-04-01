---
show: step
version: 1.0
enable_checker: true
---

# tinyCoroLab4b实验解析

## tinyCoroLab4b实验解析

> ⚠️tinyCoroLab的实验强烈推荐实验者独自完成而非直接翻阅实验解析，否则这与读完题直接翻看参考答案无太大区别，实验解析仅供实验者参考。

本节将会以tinyCoroLab的官方实现[tinyCoro](https://github.com/sakurs2/tinyCoro)为例，为大家分析并完成lab4b，请实验者预先下载tinyCoro的代码到本地。

```shell
git clone https://github.com/sakurs2/tinyCoro
```

并打开[include/coro/comp/latch.hpp](https://github.com/sakurs2/tinyCoroLab/blob/master/include/coro/comp/latch.hpp)并大致浏览代码结构。

## 📖lab4b任务参考实现

### 🧑‍💻Task #1 - 实现latch

在event的基础上实现latch是非常简单的，tinyCoro的实现如下：

```cpp
class latch
{
public:
    using event_t = event<>;
    latch(std::uint64_t count) noexcept : m_count(count), m_ev(count <= 0) {}
    latch(const latch&)                    = delete;
    latch(latch&&)                         = delete;
    auto operator=(const latch&) -> latch& = delete;
    auto operator=(latch&&) -> latch&      = delete;

    auto count_down() noexcept -> void
    {
        if (m_count.fetch_sub(1, std::memory_order::acq_rel) <= 1)
        {
            m_ev.set();
        }
    }

    auto wait() noexcept -> event_t::awaiter { return m_ev.wait(); }

private:
    std::atomic<std::int64_t> m_count;
    event_t                   m_ev;
};
```

latch内部维护一个计数，由于是多线程所以采用原子变量实现，其对协程的挂载与恢复机制是完全通过event完成的，对于`wait`直接返回event的`wait`，而对于`count_down`，只需要在计数降至0后对event执行set就好了。

## 实验总结

- 通过利用event实现latch并学会了如何有效利用现有资源简化新功能的添加并减少冗余代码逻辑
