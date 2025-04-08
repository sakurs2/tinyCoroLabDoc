---
show: step
version: 1.0
enable_checker: true
---
# tinyCoroLab4d: 构建基础协程同步组件mutex

## tinyCoroLab4d实验简介

本节我们将正式开始tinyCoroLab4d，即构建最为重要的基础协程同步组件mutex，相较于lab4前三个实验，本节实验难度略微提高。

#### 预备知识

> ⚠️预备知识即在实验开始前你应该已经掌握的知识，且在[知识铺垫章节]()中均有涉及

- **C++协程awaiter的概念**
- **C++模板编程**

## 📖lab4d任务书

### 实验前置讲解

mutex作为概念上最为人熟知、使用上最为广泛也最为重要的线程同步手段，我想实验者一定不会陌生，而tinyCoro的协程同步组件大家庭也一定不会缺席这位最重要的成员。

本节实验涉及到的核心文件为[include/coro/comp/mutex.hpp](https://github.com/sakurs2/tinyCoroLab/blob/master/include/coro/comp/mutex.hpp)和[src/comp/mutex.cpp](https://github.com/sakurs2/tinyCoroLab/blob/master/src/comp/mutex.cpp)，实验者需要预先打开文件浏览大致代码结构，下面针对该文件内容进行讲解。

[include/coro/comp/event.hpp](https://github.com/sakurs2/tinyCoroLab/blob/master/include/coro/comp/event.hpp)中给出了一个非常简单的mutex的定义，注意该定义仅仅是一个形式，不具备mutex的正确功能，但是**其类以及函数声明形式是正确的**。

另外在[include/coro/comp/mutex_guard.hpp](https://github.com/sakurs2/tinyCoroLab/blob/master/include/coro/comp/mutex_guard.hpp)中定义了`lock_guard`用来自动加锁和释放锁，这部分实验者不需要修改。

### ⚠️注意事项

- 请确保已阅读过**tinyCoroLab Introduce**章节。
- 为了确保正确实现目标函数，实验者可能需要做一些额外操作：新增类、修改现有类的实现、补充现有类的方法和成员变量等操作，请遵循**free-design实验原则**。
- 你需要仔细评估待实现的接口是否需要是线程安全的。
- 任何导致测试卡住、崩溃等无法使测试顺利通过的情况都表明你的代码存在问题。

### 实验任务书

#### 🧑‍💻Task #1 - 实现mutex

##### 任务目标

tinyCoro的mutex功能与C++的std::mutex功能类似，其核心功能是确保多个协程对共享变量操作的安全性，lab4d要求实现者为mutex添加下列功能：

```cpp
auto try_lock() noexcept -> bool; // 普通调用
auto lock() noexcept ->awaiter; // 协程调用，awaiter的await_resume返回void
auto unlock() noexcept -> void; // 普通调用
auto lock_guard() noexcept -> awaiter; // 协程调用，awaiter的await_resume返回lock_guard
```

`try_lock`即尝试获取锁，返回值表示是否成功获取锁。`lock`作为协程调用需要通过`co_await mutex.lock()`的形式调用。`unlock`即释放锁，并唤醒一个suspend awaiter（如果存在的话）。`lock_guard`封装了一系列复合操作，通过`co_await mutex.lock_guard()`的形式调用来获取锁并返回lock_guard，而lock_guard在生命周期结束后会自动释放锁。lock_guard已经实现，实验者需要做的是将其通过awaiter的await_resume返回给调用者。

对比lab4前三节实验构建mutex的难点在于调用`unlock`后如果不存在suspend awaiter那么应该将锁置于未加锁状态，如果存在那么只能唤醒一个suspend awaiter，这个过程的状态转换是需要实验者认真思考的，而且实验者**要确保mutex内部操作的线程安全性**。

这时候实验者肯定会产生一个问题：**如果`unlock`只能唤醒一个suspend awaiter，那么该唤醒哪一个呢**？这个不做要求，不管策略是FIFO还是LIFO，只需要保证所有协程可以正确的获取以及释放锁就行了。

下面给出mutex的使用场景便于实验者理解：

```cpp
mutex mtx;
task<> func() { // 手动加锁和解锁
  co_await mtx.lock();
  // codes....
  mtx.unlock();
}
task<> func() { // 自动加锁和解锁
  auto guard = co_await mtx.lock_guard();
  // codes....
  // auto unlock mutex
}
```

##### 涉及文件

- [include/coro/comp/mutex.hpp](https://github.com/sakurs2/tinyCoroLab/blob/master/include/coro/comp/mutex.hpp)
- [src/comp/mutex.cpp](https://github.com/sakurs2/tinyCoroLab/blob/master/src/comp/mutex.cpp)

##### 待实现函数

- `coro::mutex::try_lock`
- `coro::mutex::lock`
- `coro::mutex::unlock`
- `coro::mutex::lock_guard`

#### 补充说明

- **你的实现必须包含任务目标中描述的函数且函数声明形式必须一致，不然无法正常编译**
- **协程函数的返回类型可以被修改，但必须是awaiter或者awaitable类型且await_resume返回类型与任务书规定一致**
- **请务必考虑多线程安全性问题**
- **因为tinyCoro设计问题，所以协程组件恢复suspend awaiter时最好将其派发到其原本运行的context**

### 🔖测试

#### 功能测试

功能测试场景主要针对：

- **mutex的lock与unlock常规测试**
- **mutex与event的混合测试**
- **mutex与latch的混合测试**
- **mutex与wait_group的混合测试**

完成本节实验后，实验者请在构建目录下执行下列指令来构建以及运行测试程序：

```shell
make build-lab4d # 构建
make test-lab4d # 运行
```

#### 内存安全测试

在构建目录下运行下列指令来执行内存安全测试：

```shell
make memtest-lab4d
```

测试通过会提示pass，不通过会给出valgrind的输出文件位置，请实验者根据该文件排查内存故障。

#### 性能测试

> 💡**tinyCoroLab**预置了用于性能调优的火焰图生成脚本哦！详情请查看[scripts/README.MD](https://github.com/sakurs2/tinyCoroLab/blob/master/scripts/README.MD)。

在**tinyCoroLab Introduce**章节中提到性能测试的三种模型：

- **thread_pool_stl_XX:** 使用简单的线程池和stl组件。
- **coro_stl_XX:** 使用coro调度器和stl组件。
- **coro_XX:** 使用coro调度器和coro组件。

对于lab4d的性能测试，coro组件即实验者实现的mutex，stl组件即C++ std::mutex，测试场景为多个函数对同一个资源`lock`与`unlock`，并测量执行总耗时。

> 实验者只要重点关注**coro_stl_XX**和**coro_XX**模型输出的结果差异即可，该结果反映的实验者的实现与stl实现的性能差异。由于线程在受线程同步组件影响而陷入阻塞态时并不会选择执行其他任务，但tinyCoro执行引擎会，因此为了保证公平性，每个线程只会被派发一个任务，性能测试也仅仅是想重点关注实验者的实现与stl实现的性能差异。

在构建目录下运行下列指令来构建和运行性能测试：

```shell
make benchbuild-lab4d # 构建
make benchtest-lab4d # 运行
```
