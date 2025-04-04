---
show: step
version: 1.0
enable_checker: true
---
# tinyCoroLab4c: 构建基础协程同步组件wait_group

## tinyCoroLab4c实验简介

本节我们将正式开始tinyCoroLab4c，即构建基础协程同步组件wait_group。如果实验者对latch进行了正确的实现，那么wait_group的构建将会十分轻松。

#### 预备知识

> ⚠️预备知识即在实验开始前你应该已经掌握的知识，且在[知识铺垫章节]()中均有涉及

- **C++协程awaiter的概念**
- **C++模板编程**

## 📖lab4c任务书

### 实验前置讲解

本节实验涉及到的核心文件为[include/coro/comp/wait_group.hpp](https://github.com/sakurs2/tinyCoroLab/blob/master/include/coro/comp/wait_group.hpp)和[src/comp/wait_group.cpp](https://github.com/sakurs2/tinyCoroLab/blob/master/src/comp/wait_group.cpp)，实验者需要预先打开文件浏览大致代码结构，下面针对该文件内容进行讲解。

与lab4a情况类似，[include/coro/comp/wait_group.hpp](https://github.com/sakurs2/tinyCoroLab/blob/master/include/coro/comp/wait_group.hpp)中给出了一个非常简单的wait_group的定义，注意该定义仅仅是一个形式，不具备wait_group的正确功能，但是**其类以及函数声明形式是正确的**。

然后…………没了🤡，因为本节实验的大部分内容你在lab4b中已经完成了。

### ⚠️注意事项

- 请确保已阅读过**tinyCoroLab Introduce**章节。
- 为了确保正确实现目标函数，实验者可能需要做一些额外操作：新增类、修改现有类的实现、补充现有类的方法和成员变量等操作，请遵循**free-design实验原则**。
- 你需要仔细评估待实现的接口是否需要是线程安全的。
- 任何导致测试卡住、崩溃等无法使测试顺利通过的情况都表明你的代码存在问题。

### 实验任务书

#### 🧑‍💻Task #1 - 实现wait_group

##### 任务目标

实验者可能会疑惑，前几节实验涉及的协程同步组件均可以在C++中找到原型，但wait_group好像真的没有听说过？因为这是来自golang中的同步组件，但本身用法与std::latch大致相同，首先看一下实验者需要为wait_group实现的函数声明：

```cpp
auto add(int count) noexcept -> void; // 普通调用
auto done() noexcept -> void; // 普通调用
auto wait() noexcept -> awaiter; // 协程调用，awaiter的await_resume返回void
```

wait_group与latch相似本身同样持有计数并在构造函数里指明初始计数，`done`与latch的`count_down`功能完全一致，`wait`与latch的`wait`功能也完全一致，但不同的是wait_group多了一个`add`，可以为wait_group增加引用计数，这就表明即使wait_group的计数在降至0但通过`add`增加，那么后续的`wait_func`在执行`co_await wait_group.wait()`时依然会陷入suspend状态。

**有实验者可能会问在功能上latch似乎是wait_group的子集，只要wait_group不就可以了**？原因主要是设计latch是想与C++中的std::latch保持一致，但latch初始计数只能在构造函数指定且不能增加，而wait_group则适用于可能并不清楚计数具体该多少的场景，比如利用wait_group等待子任务完成，此时子任务的数量可能无法获取，但每次接收执行一个子任务均可以对计数加1，这样就能正确等待子任务完成了。

另外，考虑到wait_group的计数可以被增加，那么实验者可能会问：**会不会出现初始计数为`n`但`done`执行了`n+1`次导致计数为-1，此时通过`add`又将计数置为0，那么这个过程中唤醒suspend awaiter的逻辑是什么**？在golang的规定中`done`的调用次数超过计数属于用户使用错误，会导致预期外的行为，实验者同样可以这样设计，但在tinyCoroLab的测试设计原则中有一条是**理智**，即tinyCoroLab的测试是基于理智用户的行为，所以测试不会出现让计数降至小于0的情况，实验者也可忽略这种情况的处理办法。

下面给出wait_group的使用场景便于实验者理解：

```cpp
wait_group wg;
wg.add(1);
task<> done_func() {
  // codes...
  wg.done();
}
task<> wait_func() {
  co_await wg.wait();
  // codes...
}
```

##### 涉及文件

- [include/coro/comp/wait_group.hpp](https://github.com/sakurs2/tinyCoroLab/blob/master/include/coro/comp/wait_group.hpp)
- [src/comp/wait_group.cpp](https://github.com/sakurs2/tinyCoroLab/blob/master/src/comp/wait_group.cpp)

##### 待实现函数

- `coro::wait_group::add`
- `coro::wait_group::done`
- `coro::wait_group::wait`

#### 补充说明

- **你的实现必须包含任务目标中描述的函数且函数声明形式必须一致，不然无法正常编译**
- **协程函数的返回类型可以被修改，但必须是awaiter或者awaitable类型且await_resume返回类型与任务书规定一致**
- **请务必考虑多线程安全性问题**
- **因为tinyCoro设计问题，所以协程组件恢复suspend awaiter时最好将其派发到其原本运行的context**

### 🔖测试

#### 功能测试

功能测试场景主要针对：

- **单个context下wait_group的add、done与wait**
- **多个context下wait_group的add、done与wait**

完成本节实验后，实验者请在构建目录下执行下列指令来构建以及运行测试程序：

```shell
make build-lab4c # 构建
make test-lab4c # 运行
```

#### 内存安全测试

在构建目录下运行下列指令来执行内存安全测试：

```shell
make memtest-lab4c
```

测试通过会提示pass，不通过会给出valgrind的输出文件位置，请实验者根据该文件排查内存故障。

#### 性能测试

> 💡**tinyCoroLab**预置了用于性能调优的火焰图生成脚本哦！详情请查看[scripts/README.MD](https://github.com/sakurs2/tinyCoroLab/blob/master/scripts/README.MD)。

在**tinyCoroLab Introduce**章节中提到性能测试的三种模型：

- **thread_pool_stl_XX:** 使用简单的线程池和stl组件。
- **coro_stl_XX:** 使用coro调度器和stl组件。
- **coro_XX:** 使用coro调度器和coro组件。

对于lab4c的性能测试，coro组件即实验者实现的wait_group，stl组件虽然没有wait_group，但仍然可以使用C++ std::latch代替，测试场景为多个函数done和多个函数wait，并测量执行总耗时。

> 实验者只要重点关注**coro_stl_XX**和**coro_XX**模型输出的结果差异即可，该结果反映的实验者的实现与stl实现的性能差异。由于线程在受线程同步组件影响而陷入阻塞态时并不会选择执行其他任务，但tinyCoro执行引擎会，因此为了保证公平性，每个线程只会被派发一个任务，性能测试也仅仅是想重点关注实验者的实现与stl实现的性能差异。

在构建目录下运行下列指令来构建和运行性能测试：

```shell
make benchbuild-lab4c # 构建
make benchtest-lab4c # 运行
```
