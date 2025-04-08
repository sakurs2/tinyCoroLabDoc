---
show: step
version: 1.0
enable_checker: true
---
# tinyCoroLab5b: 构建进阶协程同步组件condition_variable

## tinyCoroLab5b实验简介

本节我们将正式开始tinyCoroLab5b，即构建进阶协程同步组件condition_variable，本节的实验涉及到mutex的使用，请实验者确保lab4d已完成。

#### 预备知识

> ⚠️预备知识即在实验开始前你应该已经掌握的知识，且在[知识铺垫章节]()中均有涉及

- **C++协程awaiter的概念**

## 📖lab5b任务书

### 实验前置讲解

本节实验涉及到的核心文件为[include/coro/comp/condition_variable.hpp](https://github.com/sakurs2/tinyCoroLab/blob/v1.0/include/coro/comp/condition_variable.hpp)和[src/comp/condition_variable.cpp](https://github.com/sakurs2/tinyCoroLab/blob/v1.0/src/comp/condition_variable.cpp)，实验者需要预先打开文件浏览大致代码结构，下面针对该文件内容进行讲解。

tinyCoro中的condition_variable与C++的std::condition_variable功能是一样的，因此我们首先需要回顾一下std::condition_variable的用法。

- **wait：** 阻塞当前线程，直到另一个线程调用同一个std:condition_variable实例的notify_one或notify_all方法，或者直到指定的谓词函数(即条件)返回 true。如果没有收到通知，即使条件为true也不会继续执行。如果收到通知了，但是条件不成立将仍然被阻塞，不会执行。
- **notify_one：** 唤醒所有等待该条件变量的线程，通常用于广播通知。
- **notify_all：** 唤醒一个等待该条件变量的线程，通常用于优先级调度或队列处理。

上述只是列出了部分方法，但对于实现lab5b已经够了，[include/coro/comp/condition_variable.hpp](https://github.com/sakurs2/tinyCoroLab/blob/v1.0/include/coro/comp/condition_variable.hpp)中给出了一个非常简单的condition_variable的定义，注意该定义仅仅是一个形式，不具备condition_variable的正确功能，但是**其类以及函数声明形式是正确的**。

需要额外注意的是std::condition_variable可以与std::mutex搭配，因此coro::std::condition_variable也会与coro::mutex搭配。

### ⚠️注意事项

- 请确保已阅读过**tinyCoroLab Introduce**章节。
- 为了确保正确实现目标函数，实验者可能需要做一些额外操作：新增类、修改现有类的实现、补充现有类的方法和成员变量等操作，请遵循**free-design实验原则**。
- 你需要仔细评估待实现的接口是否需要是线程安全的。
- 任何导致测试卡住、崩溃等无法使测试顺利通过的情况都表明你的代码存在问题。

### 实验任务书

#### 🧑‍💻Task #1 - 实现condition_variable

##### 任务目标

实验前置讲解中提到的std::condition_variable的部分方法便是我们需要在lab5b完成的方法，其具体功能不再赘述，lab5b要求实验者为condition_variable实现的功能如下：

```cpp
auto wait(mutex& mtx) noexcept -> awaiter; // 协程调用，awaiter的await_resume返回void
auto wait(mutex& mtx, cond_type&& cond) noexcept -> awaiter; // 协程调用，awaiter的await_resume返回void
auto wait(mutex& mtx, cond_type& cond) noexcept -> awaiter; // 协程调用，awaiter的await_resume返回void
auto notify_one() noexcept -> void; // 普通调用
auto notify_all() noexcept -> void; // 普通调用
```

对于`wait`，第一个参数为tinyCoro的mutex，并且可以选择附带一个条件谓词参数。当不带条件谓词时，协程会直接陷入suspend状态，如果带条件谓词，会先检查条件谓词的是否为true，是则恢复运行，否则陷入suspend状态，**注意这与std::condition_variable的行为不同，std::condition_variable的wait在带谓词的情况下会直接让线程阻塞，只有被唤醒后才会检查谓词的结果，实验者按照lab5b的规定来就好。**

对于`notify_one`和`notify_all`，均会唤醒因调用`condition_variable.wait`陷入suspend状态的协程，前者会唤醒一个，后者会唤醒全部，被唤醒的协程如果存在条件谓词，会检查谓词的结果，如果为true则尝试获取锁，否则继续陷入suspend状态。另外对于`notify_one`如何挑选一个协程来唤醒不作要求，只要保证正确唤醒一个就行。

> **💡调用notify_one或者notify_all一定要处于持有锁的状态吗？**
> 参照std::condition_variable的行为，持有锁和发起notify没有什么关联。
> **💡发起notify但没有陷入suspend的协程怎么办？会使得之后将要陷入suspend状态的协程直接恢复吗？**
> 参照std::condition_variable的行为，通知行为不会累加，没有陷入suspend的协程那就忽略，之后将要陷入suspend状态的协程会被之后的notify唤醒。
<!-- > 💡两个协程一个发起notify一个发起wait，需要在`condition_variable`中作某些同步处理 -->

如果你对lab5b的condition_variable的某些行为还存在疑惑，那么参照std::condition_variable怎么做就可以了，唯一的不同点刚才已经提过。

下面给出condition_variable的使用场景便于实验者理解：

```cpp
// 场景一：保证全局操作有序性
condition_variable cv;
mutex mtx;
int global_id;
task<> func(int id) {
  auto guard = co_await mtx.lock_guard();
  co_await cv.wait(mtx, [&](){id==global_id;});
  global_id+=1;
  cv.notify_all();
}
// 场景二：协程安全的多生产者多消费者队列
class queue{
public:
  task<> push(int number) {
    auto guard = co_await mtx.lock_guard();
    co_await producer_cv.wait(mtx, [&](){que.size()<capacity;});
    que.push_back(number);
    consumer_cv.notify_one();
  }

  task<int> pop() {
    auto guard = co_await mtx.lock_guard();
    co_await consumer_cv.wait(mtx, [&](){!que.empty();});
    auto number = que.front();
    que.pop();
    producer_cv.notify_one();
    co_return number;
  }

private:
  const int capacity;
  condition_variable producer_cv;
  condition_variable consumer_cv;
  mutex mtx;
  std::queue<int> que;
};

```

从上面的例子中可以看出在使用上tinyCoro的condition_variable与std::condition_variable几乎一致，实验者最好根据具体使用场景仔细评估自己的设计方案是否正确。

##### 涉及文件

- [include/coro/comp/condition_variable.hpp](https://github.com/sakurs2/tinyCoroLab/blob/v1.0/include/coro/comp/condition_variable.hpp)
- [src/comp/condition_variable.cpp](https://github.com/sakurs2/tinyCoroLab/blob/v1.0/src/comp/condition_variable.cpp)

##### 待实现函数

- `coro::condition_variable::wait`
- `coro::condition_variable::notify_one`
- `coro::condition_variable::notify_all`

#### 补充说明

- **你的实现必须包含任务目标中描述的函数且函数声明形式必须一致，不然无法正常编译**
- **协程函数的返回类型可以被修改，但必须是awaiter或者awaitable类型且await_resume返回类型与任务书规定一致**
- **请务必考虑多线程安全性问题**
- **因为tinyCoro设计问题，所以协程组件恢复suspend awaiter时最好将其派发到其原本运行的context**

### 🔖测试

#### 功能测试

功能测试场景主要针对：

- **利用condition_variable保证协程执行的有序性**
- **利用condition_variable构建生产者和消费者模型**

完成本节实验后，实验者请在构建目录下执行下列指令来构建以及运行测试程序：

```shell
make build-lab5b # 构建
make test-lab5b # 运行
```

#### 内存安全测试

在构建目录下运行下列指令来执行内存安全测试：

```shell
make memtest-lab5b
```

测试通过会提示pass，不通过会给出valgrind的输出文件位置，请实验者根据该文件排查内存故障。

#### 性能测试

> 💡**tinyCoroLab**预置了用于性能调优的火焰图生成脚本哦！详情请查看[scripts/README.MD](https://github.com/sakurs2/tinyCoroLab/blob/v1.0/scripts/README.MD)。

在**tinyCoroLab Introduce**章节中提到性能测试的三种模型：

- **thread_pool_stl_XX:** 使用简单的线程池和stl组件。
- **coro_stl_XX:** 使用coro调度器和stl组件。
- **coro_XX:** 使用coro调度器和coro组件。

对于lab5b的性能测试，coro组件即实验者实现的condition_variable和mutex，stl组件即C++ std::condition_variable和std::mutex，测试场景分为利用condition_variable保证执行的有序性和利用condition_variable构建生产者和消费者模型，并测量执行总耗时。

> 实验者只要重点关注**coro_stl_XX**和**coro_XX**模型输出的结果差异即可，该结果反映的实验者的实现与stl实现的性能差异。由于线程在受线程同步组件影响而陷入阻塞态时并不会选择执行其他任务，但tinyCoro执行引擎会，因此为了保证公平性，每个线程只会被派发一个任务，性能测试也仅仅是想重点关注实验者的实现与stl实现的性能差异。

在构建目录下运行下列指令来构建和运行性能测试：

```shell
make benchbuild-lab5b # 构建
make benchtest-lab5b # 运行
```
