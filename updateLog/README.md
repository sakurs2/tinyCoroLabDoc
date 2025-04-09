# 🔥更新日志

记录tinyCoroLab课程全部资源的版本更新日志

> **⚠️tinyCoro、tinyCoroLab和tinyCoroLabDoc的版本号命名一致，实验者在做某版本的tinyCoroLab时请切换至对应版本的tinyCoroLabDoc**

<!-- # 更新日志

## [1.0.0] - 2024-06-15
### 新增功能
- 添加了用户认证功能，支持登录和注册。
- 引入了新的 API 接口，用于数据查询和管理。
- 增加了多语言支持，用户可以选择不同的语言界面。

### 修复问题
- 修复了登录页面在某些浏览器下显示不正确的问题。
- 修复了数据同步时可能出现的死锁问题。
- 修复了用户在上传大文件时的性能问题。

### 性能改进
- 优化了数据库查询，提升了数据加载速度。
- 压缩了静态资源文件，减少了页面加载时间。
- 优化了服务器配置，提升了整体响应速度。

### 其他变更
- 更新了项目文档，增加了新的使用指南和示例。
- 调整了用户界面，提升了用户体验。
- 增加了新的测试用例，确保代码质量。 -->
## [tinyCoro](https://github.com/sakurs2/tinyCoro)

#### [v1.1] - 2025-04-09

##### 主要变更

1.0版本scheduler的设计存在诸多缺陷，在短期运行工作模式下context完成所有任务后会立刻停止（因为scheduler会在context启动后立刻调用`notify_stop`通知其停止，context收到该信号在处理完所有任务后立刻停止），这导致context运行中无法安全的将任务派发给scheduler，因为scheduler可能会把任务派发给已经停止的context，总之，这种设计很怪异且不合理。

1.1版本移除了工作模式这一概念，重新设计了scheduler对context的交互逻辑，现在用户使用tinyCoro更简单了：

```cpp
schduler::init();
submit_to_scheduler(func());
// repeat submit...
// scheduler::start(); don;t need this
scheduler::loop();
```

在提交完所有任务后只需要调用`scheduler::loop`，**scheduler便会启动所有context然后等待context全部完成任务后再统一下发送停止信号，这样哪怕某个context先运行完毕也会处于阻塞态而不是停止，因为未来scheduler很可能向其派发新的任务**，基于此，用户可以放心的在协程任务中调用`submit_to_scheduler`。

除此外，所有测试以及样例代码中scheduler的运行全部采用上述代码逻辑。

#### [v1.0] - 2025-04-01

##### 主要变更

- 1.0版本正式发布

## [tinyCoroLab](https://github.com/sakurs2/tinyCoroLab)

#### [v1.1] - 2025-04-09

##### 主要变更

为lab2b新增了一个子任务用来实现tinyCoro1.1版本新增的功能，并为lab2b新增测试样例。

#### [v1.0] - 2025-04-01

##### 主要变更

- 1.0版本正式发布

## [tinyCoroLabDoc](https://github.com/sakurs2/tinyCoroLabDoc)

#### [v1.1] - 2025-04-09

##### 主要变更

为lab2b新增的子任务添加任务书。

#### [v1.0] - 2025-04-01

##### 主要变更

- 1.0版本正式发布
