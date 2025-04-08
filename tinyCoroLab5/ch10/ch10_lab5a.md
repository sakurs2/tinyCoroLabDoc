---
show: step
version: 1.0
enable_checker: true
---
# tinyCoroLab5a: 构建进阶协程同步组件when_all（选做）

## tinyCoroLab5a实验简介

本节我们将正式开始tinyCoroLab5a，注意！不再是构建基础协程同步组件，而是进阶协程同步组件！！！既然是进阶，那么实现上可能会有一丢丢小复杂，而本节将要实现的是一个协程函数when_all，注意本节实验属于选做，因为tinyCoro在when_all的设计上存在缺陷，所以能用但比较鸡肋，后续会重构此部分，但从学习知识的角度讲本节实验还是推荐实验者完成的。

#### 预备知识

> ⚠️预备知识即在实验开始前你应该已经掌握的知识，且在[知识铺垫章节]()中均有涉及

- **C++协程awaiter的概念**
- **C++模板编程**
- **C++ concepts**

## 📖lab5a任务书

### 实验前置讲解

本节实验涉及到的核心文件为[include/coro/comp/when_all.hpp](https://github.com/sakurs2/tinyCoroLab/blob/v1.0/include/coro/comp/when_all.hpp)，实验者需要预先打开文件浏览大致代码结构，下面针对该文件内容进行讲解。

该文件预先定义了一个when_all函数可以接收多个参数，并用concepts约束参数列表中的每个参数必须是awaitable类型，因此when_all的作用便是等待参数列表中的所有awaitable执行完毕再返回。

该文件内定义的awaiter只是为了让项目编译通过，不具有实际意义，实验者需要实现自己需要的awaiter。

### ⚠️注意事项

- 请确保已阅读过**tinyCoroLab Introduce**章节。
- 为了确保正确实现目标函数，实验者可能需要做一些额外操作：新增类、修改现有类的实现、补充现有类的方法和成员变量等操作，请遵循**free-design实验原则**。
- 你需要仔细评估待实现的接口是否需要是线程安全的。
- 任何导致测试卡住、崩溃等无法使测试顺利通过的情况都表明你的代码存在问题。

### 实验任务书

#### 🧑‍💻Task #1 - 实现event

##### 任务目标

when_all的参数实验者可以理解为是一个个待完成的协程任务，其具体行为受到参数列表的影响。

当参数列表的awaitable类型的await_resume全部返回void时，那么**when_all的作用是运行所有awaitable并等待所有awaitable返回，其返回的awaiter的await_resume函数返回类型为void。**

当参数列表的awaitable类型的await_resume全部非void且类型一致时，那么**when_all的作用同样是运行所有awaitable并等待所有awaitable返回，但其返回的awaiter的await_resume函数返回类型不再是void，而是一个支持C++ Range-based for loop语法的容器，即when_all会收集参数awaitable的返回值并以容器形式返回，但存储顺序不做要求。**

**实验者肯定会问如果参数列表的awaitable类型的await_resume返回类型不一致怎么办？** tinyCoro的实现是利用concepts对参数约束，如果出现这种情况会在编译期报错，即提醒用户这样的做法不对，当然实验者不必考虑这种情况了，至少测试中when_all参数列表的所有awaitable其await_resume返回类型都是一致的。

**怎么理解when_all需要运行所有awaitable参数呢？是在when_all里一个个执行吗？** 这里的运行指的是将其派发到执行引擎中，然后调用when_all的协程陷入suspend状态，但由于tinyCoro设计的限制when_all参数awaitable的运行最好是派发到与调用when_all的协程相同的context中。

> **💡将参数awaitable派发到与调用when_all的协程相同的context中，那岂不是等同于顺序执行？**
> 是的，但tinyCoro正在修复此部分设计缺陷，而修复完成后你可以选择派发到别的context中，此时只需要修改用于派发任务的一行代码即可。

实验者应当能发现when_all是latch的一种使用场景，因此实验者可借助latch简化when_all的实现。

下面给出when_all的使用场景便于实验者理解：

```cpp
// when_all参数列表awaitable全部返回void
task<> empty_func() {
  // codes...
  co_return;
}
task<> when_all_func() {
  co_await when_all(empty_func(),empty_func(),empty_func());
  // codes...
}

// when_all参数列表awaitable全部返回int
task<int> return_int_func() {
  // codes...
  co_return number;
}
task<> when_all_func() {
  auto container = co_await when_all(return_int_func(),return_int_func(),return_int_func());
  for(auto&it:container) {
    // codes...
  }
}
```

##### 涉及文件

- [include/coro/comp/when_all.hpp](https://github.com/sakurs2/tinyCoroLab/blob/v1.0/include/coro/comp/when_all.hpp)

##### 待实现函数

- `coro::when_all`

#### 补充说明

- **你的实现必须包含任务目标中描述的函数且函数声明形式必须一致，不然无法正常编译**
- **协程函数的返回类型可以被修改，但必须是awaiter或者awaitable类型且await_resume返回类型与任务书规定一致**
- **请务必考虑多线程安全性问题**
- **因为tinyCoro设计问题，所以协程组件恢复suspend awaiter时最好将其派发到其原本运行的context**

### 🔖测试

#### 功能测试

功能测试场景主要针对：

- **when_all参数列表awaitable全部返回void下的功能测试**
- **when_all参数列表awaitable全部返回非void下的功能测试**

完成本节实验后，实验者请在构建目录下执行下列指令来构建以及运行测试程序：

```shell
make build-lab5a # 构建
make test-lab5a # 运行
```

#### 内存安全测试

在构建目录下运行下列指令来执行内存安全测试：

```shell
make memtest-lab5a
```

测试通过会提示pass，不通过会给出valgrind的输出文件位置，请实验者根据该文件排查内存故障。
