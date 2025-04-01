---
show: step
version: 1.0
enable_checker: true
---

# tinyCoroLa5b实验解析

## tinyCoroLab5b实验解析

> ⚠️tinyCoroLab的实验强烈推荐实验者独自完成而非直接翻阅实验解析，否则这与读完题直接翻看参考答案无太大区别，实验解析仅供实验者参考。

本节将会以tinyCoroLab的官方实现[tinyCoro](https://github.com/sakurs2/tinyCoro)为例，为大家分析并完成lab5b，请实验者预先下载tinyCoro的代码到本地。

```shell
git clone https://github.com/sakurs2/tinyCoro
```

并打开[include/coro/comp/condition_variable.hpp](https://github.com/sakurs2/tinyCoroLab/blob/master/include/coro/comp/condition_variable.hpp)和[src/comp/condition_variable.cpp](https://github.com/sakurs2/tinyCoroLab/blob/master/src/comp/condition_variable.cpp)并大致浏览代码结构。

## 📖lab5b任务参考实现

### 🧑‍💻Task #1 - 实现condition_variable
