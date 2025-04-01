---
show: step
version: 1.0
enable_checker: true
---
# tinyCoroLab5c: 构建进阶协程同步组件channel

## tinyCoroLab5c实验简介

本节我们将正式开始tinyCoroLab5c，即构建进阶协程同步组件channel。

#### 预备知识

> ⚠️预备知识即在实验开始前你应该已经掌握的知识，且在[知识铺垫章节]()中均有涉及

- **C++协程awaiter的概念**

## 📖lab5c任务书

### 实验前置讲解

本节实验涉及到的核心文件为[include/coro/comp/channel.hpp](https://github.com/sakurs2/tinyCoroLab/blob/master/include/coro/comp/channel.hpp)和[src/comp/channel.cpp](https://github.com/sakurs2/tinyCoroLab/blob/master/src/comp/channel.cpp)，实验者需要预先打开文件浏览大致代码结构，下面针对该文件内容进行讲解。

[include/coro/comp/channel.hpp](https://github.com/sakurs2/tinyCoroLab/blob/master/include/coro/comp/channel.hpp)中给出了一个非常简单的channel的定义，注意该定义仅仅是一个形式，不具备channel的正确功能，但是**其类以及函数声明形式是正确的**。

### ⚠️注意事项

- 请确保已阅读过**tinyCoroLab Introduce**章节。
- 为了确保正确实现目标函数，实验者可能需要做一些额外操作：新增类、修改现有类的实现、补充现有类的方法和成员变量等操作，请遵循**free-design实验原则**。
- 你需要仔细评估待实现的接口是否需要是线程安全的。
- 任何导致测试卡住、崩溃等无法使测试顺利通过的情况都表明你的代码存在问题。

### 实验任务书

#### 🧑‍💻Task #1 - 实现channel

##### 任务目标

channel与wait_group一样也是来自golang的协程同步组件，其功能可以类比为一个容量固定的阻塞式调用的多生产者多消费者队列。channel的定义包含两个模板参数：存储类型T和容量capacity，且容量默认为1。

lab5c要求实验者为channel实现的功能如下：

```cpp
template<typename value_type>
        requires(std::is_constructible_v<T, value_type &&>)
auto send(value_type&& value) noexcept -> awaiter; // 协程调用，awaiter的await_resume返回bool
auto recv() noexcept -> awaiter; // 协程调用，awaiter的，awaiter的await_resume返回std::optional<T>
auto close() noexcept -> void; // 普通调用
```

channel可以有多个生产者和多个消费者，对于所有生产者生产的值都应该被正确存储，对于多个消费者，lab5c不会规定具体怎么分配给值给这些消费者，这由实验者决定，只需要确保所有生产过的值会被消费且仅被消费一次。

如果channel容量已满，那么生产者协程将陷入suspend状态，此时消费者消费数据后将会唤醒一个或多个生产者协程（实验者自行决定）。如果channel存储数据为空，那么消费者协程将陷入suspend状态，此时生产者生产数据将会唤醒一个或多个消费者协程。

当对channel调用close后，所有陷入suspend状态的生产者和消费者都需要被唤醒，此时生产者调用`send`会返回false，表明channel已经关闭，而消费者仍然可以消费channel中剩余的数据，直到容量为空后消费者调用`recv`返回的std::optional不再具有值，以此判断channel是否关闭，这就是为何选用std::optional作为返回值的原因，**实验者实现的`recv`也必须返回std::optional。**

在lab5b中实验者不难发现condition_variable也可以构建与channel类似功能的数据结构，因此实验者可以利用condition_variable简化channel的实现。

另外关于实现效率上，**实验者应该考虑区分send数据的左右值并在存储以及返回数据时考虑是用拷贝语义还是移动语义，** 性能测试会涉及长字符串的生产和消费。

下面给出channel的一种使用场景便于实验者加深理解：

```cpp
// 利用channel构建多生产者多消费者模型
channel<int> ch;
latch lt(producer_number);
task<> producer() {
  for(int i=0;i<loop_num;i++) {
    co_await ch.send(data); // 在该场景下send返回值可以忽略
  }
  lt.count_down();
}
task<> consumer() {
  while(true) {
    auto data = co_await ch.recv();
    if(!data) {
      break;
    }
    // codes...
  }
}
task<> close() {
  co_await lt.wait();
  ch.close(); // 保证生产者全部生产完毕后再关闭channel
}
```

##### 涉及文件

- [include/coro/comp/channel.hpp](https://github.com/sakurs2/tinyCoroLab/blob/master/include/coro/comp/channel.hpp)
- [src/comp/channel.cpp](https://github.com/sakurs2/tinyCoroLab/blob/master/src/comp/channel.cpp)

##### 待实现函数

- `coro::channel::send`
- `coro::channel::recv`
- `coro::channel::close`

#### 补充说明

- **你的实现必须包含任务目标中描述的函数且函数声明形式必须一致，不然无法正常编译**
- **协程函数的返回类型可以被修改，但必须是awaiter或者awaitable类型且await_resume返回类型与任务书规定一致**
- **请务必考虑多线程安全性问题**
- **因为tinyCoro设计问题，所以协程组件恢复suspend awaiter时最好将其派发到其原本运行的context**

### 🔖测试

#### 功能测试

功能测试场景主要针对：

- **利用channel构建的生产者和消费者模型**

需要注意的不管channel容量为多大，测试均不会在过程中检查其内部状态，只会在所有协程任务结束后检查channel是否满足所有被生产的数据均被消费且仅消费一次。

完成本节实验后，实验者请在构建目录下执行下列指令来构建以及运行测试程序：

```shell
make build-lab5c # 构建
make test-lab5c # 运行
```

#### 内存安全测试

在构建目录下运行下列指令来执行内存安全测试：

```shell
make memtest-lab5c
```

测试通过会提示pass，不通过会给出valgrind的输出文件位置，请实验者根据该文件排查内存故障。

#### 性能测试

> 💡**tinyCoroLab**预置了用于性能调优的火焰图生成脚本哦！详情请查看[scripts/README.MD](https://github.com/sakurs2/tinyCoroLab/blob/master/scripts/README.MD)。

在**tinyCoroLab Introduce**章节中提到性能测试的三种模型：

- **thread_pool_stl_XX:** 使用简单的线程池和stl组件。
- **coro_stl_XX:** 使用coro调度器和stl组件。
- **coro_XX:** 使用coro调度器和coro组件。

对于lab5c的性能测试，coro组件即实验者实现的channel，stl组件即C++ std::condition_variable和std::mutex，测试场景为单生产者和单消费者的消费模型，并测量执行总耗时。需要注意的是测试场景会对长字符串进行生产和消费，如果实验者实现的channel不考虑移动语义，那么拷贝将会产生巨大的开销。

> 实验者只要重点关注**coro_stl_XX**和**coro_XX**模型输出的结果差异即可，该结果反映的实验者的实现与stl实现的性能差异。由于线程在受线程同步组件影响而陷入阻塞态时并不会选择执行其他任务，但tinyCoro执行引擎会，因此为了保证公平性，每个线程只会被派发一个任务，性能测试也仅仅是想重点关注实验者的实现与stl实现的性能差异。

在构建目录下运行下列指令来构建和运行性能测试：

```shell
make benchbuild-lab5c # 构建
make benchtest-lab5c # 运行
```
