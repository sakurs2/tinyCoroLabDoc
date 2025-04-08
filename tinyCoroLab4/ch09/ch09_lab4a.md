---
show: step
version: 1.0
enable_checker: true
---
# tinyCoroLab4a: 构建基础协程同步组件event

## tinyCoroLab4a实验简介

本节我们将正式开始tinyCoroLab4a，即构建基础协程同步组件event。在开始本节实验前请实验者务必确保已透彻理解lab4 pre所讲的内容。

#### 预备知识

> ⚠️预备知识即在实验开始前你应该已经掌握的知识，且在[知识铺垫章节]()中均有涉及

- **C++协程awaiter的概念**
- **C++模板编程**

## 📖lab4a任务书

### 实验前置讲解

本节实验涉及到的核心文件为[include/coro/comp/event.hpp](https://github.com/sakurs2/tinyCoroLab/blob/v1.0/include/coro/comp/event.hpp)和[src/comp/event.cpp](https://github.com/sakurs2/tinyCoroLab/blob/v1.0/src/comp/event.cpp)，实验者需要预先打开文件浏览大致代码结构，下面针对该文件内容进行讲解。

[include/coro/comp/event.hpp](https://github.com/sakurs2/tinyCoroLab/blob/v1.0/include/coro/comp/event.hpp)中给出了一个非常简单的event的定义，注意该定义仅仅是一个形式，不具备event的正确功能，但是**其类以及函数声明形式是正确的**。

另外该文件预定义了`event_guard`，在其生命周期结束后可以自动对event调用set，但仅限于模板参数为空的event。实验者不要修改任何关于guard的定义。

### ⚠️注意事项

- 请确保已阅读过**tinyCoroLab Introduce**章节。
- 为了确保正确实现目标函数，实验者可能需要做一些额外操作：新增类、修改现有类的实现、补充现有类的方法和成员变量等操作，请遵循**free-design实验原则**。
- 你需要仔细评估待实现的接口是否需要是线程安全的。
- 任何导致测试卡住、崩溃等无法使测试顺利通过的情况都表明你的代码存在问题。

### 实验任务书

#### 🧑‍💻Task #1 - 实现event

##### 任务目标

tinyCoro的event与C++中的std::promise功能类似，带有一个返回类型模板参数且默认为空，并且提供`set`以及`wait`两个接口，`set`可以直接调用，但调用`wait`需要是协程的形式`co_await wait()`。

对于模板参数为空的event，其功能类似于传递信号，其核心函数声明如下：

```cpp
auto set() noexcept -> void; // 普通调用
auto wait() noexcept -> awaiter; // 协程调用，awaiter的await_resume返回void
```

> 💡**什么是普通调用和协程调用？**
> 普通调用就是像正常函数调用即可，协程调用需要在被调用函数前添加co_await关键字，因此协程调用只能出现在协程函数内。
> 💡**为何有些方法是普通调用而有些是协程调用？**
> 对于可能会让协程陷入suspend状态的调用需要是协程调用，否则就实现为普通调用即可，不要产生不必要的协程调用开销。

对于模板参数非空的event，其功能类似于传递数据，其核心函数声明如下：

```cpp
template<typename value_type>
auto set(value_type&& value) noexcept -> void; // 普通调用
auto wait() noexcept -> awaiter; // 协程调用，awaiter的await_resume返回set设置的值
```

对其调用`co_await event.wait()`的返回值是set设置的值，注意因为考虑到隐式转换问题，set的入参被设置成模板参数`value_type`，与event的模板参数`return_type`并不同，实验者可以不使用这种模板形式，只要保证数据能被正确传递即可。

下面给出event的使用场景便于实验者理解：

```cpp
// 模板参数为空的event
event<> ev;
task<> set_func() { // 手动调用set
  // codes...
  ev.set();
}
task<> set_func() { // 自动调用set
  auto guard = event_guard(ev);
  // codes...
  // auto set ev
}
task<> wait_func() {
  co_await ev.wait();
  // codes ...
}

// 模板参数非空的event
event<int> ev;
task<> set_func() {
  // codes...
  ev.set(number);
}
task<> wait_func() {
  auto number = co_await ev.wait();
  // codes ...
}
```

##### 涉及文件

- [include/coro/comp/event.hpp](https://github.com/sakurs2/tinyCoroLab/blob/v1.0/include/coro/comp/event.hpp)
- [src/comp/event.cpp](https://github.com/sakurs2/tinyCoroLab/blob/v1.0/src/comp/event.cpp)

##### 待实现函数

- `coro::event::wait`
- `coro::event::set`

#### 补充说明

- **你的实现必须包含任务目标中描述的函数且函数声明形式必须一致，不然无法正常编译**
- **协程函数的返回类型可以被修改，但必须是awaiter或者awaitable类型且await_resume返回类型与任务书规定一致**
- **请务必考虑多线程安全性问题**
- **因为tinyCoro设计问题，所以协程组件恢复suspend awaiter时最好将其派发到其原本运行的context**

### 🔖测试

#### 功能测试

功能测试场景主要针对：

- **单个context下event的wait与set**
- **多个context下event的wait与set**

完成本节实验后，实验者请在构建目录下执行下列指令来构建以及运行测试程序：

```shell
make build-lab4a # 构建
make test-lab4a # 运行
```

#### 内存安全测试

在构建目录下运行下列指令来执行内存安全测试：

```shell
make memtest-lab4a
```

测试通过会提示pass，不通过会给出valgrind的输出文件位置，请实验者根据该文件排查内存故障。

#### 性能测试

> 💡**tinyCoroLab**预置了用于性能调优的火焰图生成脚本哦！详情请查看[scripts/README.MD](https://github.com/sakurs2/tinyCoroLab/blob/v1.0/scripts/README.MD)。

在**tinyCoroLab Introduce**章节中提到性能测试的三种模型：

- **thread_pool_stl_XX:** 使用简单的线程池和stl组件。
- **coro_stl_XX:** 使用coro调度器和stl组件。
- **coro_XX:** 使用coro调度器和coro组件。

对于lab4a的性能测试，coro组件即实验者实现的event，stl组件即C++ std::promsie，测试场景为单个函数set和多个函数wait，并测量执行总耗时。

> 实验者只要重点关注**coro_stl_XX**和**coro_XX**模型输出的结果差异即可，该结果反映的实验者的实现与stl实现的性能差异。由于线程在受线程同步组件影响而陷入阻塞态时并不会选择执行其他任务，但tinyCoro执行引擎会，因此为了保证公平性，每个线程只会被派发一个任务，性能测试也仅仅是想重点关注实验者的实现与stl实现的性能差异。

在构建目录下运行下列指令来构建和运行性能测试：

```shell
make benchbuild-lab4a # 构建
make benchtest-lab4a # 运行
```
