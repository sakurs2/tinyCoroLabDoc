---
show: step
version: 1.0
enable_checker: true
---
# tinyCoroLab4b: 构建基础协程同步组件latch

## tinyCoroLab4b实验简介

本节我们将正式开始tinyCoroLab4b，即构建基础协程同步组件latch。如果实验者对event进行了正确的实现，那么latch的构建将会十分轻松。

#### 预备知识

> ⚠️预备知识即在实验开始前你应该已经掌握的知识，且在[知识铺垫章节]()中均有涉及

- **C++协程awaiter的概念**
- **C++模板编程**

## 📖lab4b任务书

### 实验前置讲解

本节实验涉及到的核心文件为[include/coro/comp/latch.hpp](https://github.com/sakurs2/tinyCoroLab/blob/v1.0/include/coro/comp/latch.hpp)和[src/comp/latch.cpp](https://github.com/sakurs2/tinyCoroLab/blob/v1.0/src/comp/latch.cpp)，实验者需要预先打开文件浏览大致代码结构，下面针对该文件内容进行讲解。

与lab4a情况类似，[include/coro/comp/latch.hpp](https://github.com/sakurs2/tinyCoroLab/blob/v1.0/include/coro/comp/latch.hpp)中给出了一个非常简单的latch的定义，注意该定义仅仅是一个形式，不具备latch的正确功能，但是**其类以及函数声明形式是正确的**。

另外该文件预定义了`latch_guard`，在其生命周期结束后可以自动对latch调用`count_down`，实验者不要修改任何关于guard的定义。

### ⚠️注意事项

- 请确保已阅读过**tinyCoroLab Introduce**章节。
- 为了确保正确实现目标函数，实验者可能需要做一些额外操作：新增类、修改现有类的实现、补充现有类的方法和成员变量等操作，请遵循**free-design实验原则**。
- 你需要仔细评估待实现的接口是否需要是线程安全的。
- 任何导致测试卡住、崩溃等无法使测试顺利通过的情况都表明你的代码存在问题。

### 实验任务书

#### 🧑‍💻Task #1 - 实现latch

##### 任务目标

tinyCoro的latch组件与C++的std::latch功能是一样的，构造函数包含一个整数类型指明latch初始的计数，对用户提供下列两个接口：

```cpp
auto count_down() noexcept -> void; // 普通调用
auto wait() noexcept -> awaiter; // 协程调用，awaiter的await_resume返回void
```

`wait_func`调用`co_await latch.wait()`时，如果latch的计数大于0，那么陷入suspend状态，并将suspend awaiter挂载到latch里。

`count_down_func`调用`latch.count_down()`来使得计数减1，如果此时计数降至0，那么应该resume所有的suspend awaiter。

实验者不拿发现latch完全可以靠event来实现对协程的挂载以及恢复，而自身只需要维护计数就可以了。

下面给出latch的使用场景便于实验者理解：

```cpp
latch lt(1);
task<> countdown_func() { // 手动调用count_down
  // codes...
  lt.count_down();
}
task<> countdown_func() { // 自动调用count_down
  auto guard = latch_guard(lt);
  // codes...
  // auto count_down
}
task<> wait_func() {
  co_await lt.wait();
  // codes...
}
```

##### 涉及文件

- [include/coro/comp/latch.hpp](https://github.com/sakurs2/tinyCoroLab/blob/v1.0/include/coro/comp/latch.hpp)
- [src/comp/latch.cpp](https://github.com/sakurs2/tinyCoroLab/blob/v1.0/src/comp/latch.cpp)

##### 待实现函数

- `coro::latch::count_down`
- `coro::latch::wait`

#### 补充说明

- **你的实现必须包含任务目标中描述的函数且函数声明形式必须一致，不然无法正常编译**
- **协程函数的返回类型可以被修改，但必须是awaiter或者awaitable类型且await_resume返回类型与任务书规定一致**
- **请务必考虑多线程安全性问题**
- **因为tinyCoro设计问题，所以协程组件恢复suspend awaiter时最好将其派发到其原本运行的context**

### 🔖测试

#### 功能测试

功能测试场景主要针对：

- **单个context下latcth的count_down与wait**
- **多个context下latcth的count_down与wait**

完成本节实验后，实验者请在构建目录下执行下列指令来构建以及运行测试程序：

```shell
make build-lab4b # 构建
make test-lab4b # 运行
```

#### 内存安全测试

在构建目录下运行下列指令来执行内存安全测试：

```shell
make memtest-lab4b
```

测试通过会提示pass，不通过会给出valgrind的输出文件位置，请实验者根据该文件排查内存故障。

#### 性能测试

> 💡**tinyCoroLab**预置了用于性能调优的火焰图生成脚本哦！详情请查看[scripts/README.MD](https://github.com/sakurs2/tinyCoroLab/blob/v1.0/scripts/README.MD)。

在**tinyCoroLab Introduce**章节中提到性能测试的三种模型：

- **thread_pool_stl_XX:** 使用简单的线程池和stl组件。
- **coro_stl_XX:** 使用coro调度器和stl组件。
- **coro_XX:** 使用coro调度器和coro组件。

对于lab4b的性能测试，coro组件即实验者实现的latch，stl组件即C++ std::latch，测试场景为多个函数count_down和多个函数wait，并测量执行总耗时。

> 实验者只要重点关注**coro_stl_XX**和**coro_XX**模型输出的结果差异即可，该结果反映的实验者的实现与stl实现的性能差异。由于线程在受线程同步组件影响而陷入阻塞态时并不会选择执行其他任务，但tinyCoro执行引擎会，因此为了保证公平性，每个线程只会被派发一个任务，性能测试也仅仅是想重点关注实验者的实现与stl实现的性能差异。

在构建目录下运行下列指令来构建和运行性能测试：

```shell
make benchbuild-lab4b # 构建
make benchtest-lab4b # 运行
```
