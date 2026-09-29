# 5.1 ArkTS 并发模型：Actor、TaskPool 与 Worker

## ArkTS 为什么选了 Actor 模型

传统多线程编程是"内存共享 + 锁"：所有线程看到同一个堆，靠锁保证不同时改同一块数据。这套模型的坑所有人都踩过——漏加锁就是数据竞争，锁加错了就是死锁，问题还极难复现。

ArkTS 的选择是 **Actor 并发模型**（基于消息通信）：每个并发实例（主线程、TaskPool 工作线程、Worker 线程）有自己独立的虚拟机环境和独立的堆，线程之间不共享对象，要传数据就发消息——消息内容默认经过**序列化**拷贝到对端的堆里。官方并发概述文档的说法：TaskPool 和 Worker 都基于 Actor 并发模型实现，开发者无需处理锁带来的复杂问题。

代价也明明白白：传对象就是一次深拷贝（Structured Clone），数据越大越贵。官方 Sendable 文档给过一个量级参考：**100KB 的数据跨并发实例传输耗时约 1ms**。传个配置对象无所谓，传一张解码后的大图就是灾难——这个矛盾正是 5.2 节 Sendable 要解决的。

## TaskPool：调度器 + 工作线程池

TaskPool 是面向"任务"的并发设施：把函数丢进去，系统找线程执行，结果 Promise 返回。官方 TaskPool 简介对它的内部结构描述得很具体：

- **宿主线程提交任务到任务队列，系统选择合适的工作线程执行**；
- **默认只启动一个工作线程，任务多了自动扩容**；工作线程数量上限由设备的物理核数决定，内部管理具体数量；长时间无任务分发时缩容；
- **支持优先级**：HIGH、MEDIUM、LOW、IDLE，高优先级任务在资源不足时更容易获得系统资源；为防止低优先级任务饿死，等待过久的任务会被提升调度机会（防饥饿机制）[待验证：优先级提升的精确算法，官方未公开细节]；
- 支持任务取消（`taskpool.Cancel`）、任务组（TaskGroup，一组关联任务统一等待，API 24 起支持给任务组配超时和优先级）。

用法是把 `@Concurrent` 装饰的函数提交进去：

```typescript
import { taskpool } from '@kit.ArkTS';

@Concurrent
function histogram(pixels: ArrayBuffer): number[] {
  // CPU 密集计算，运行在 TaskPool 工作线程
  return compute(pixels);
}

async function run() {
  let task = new taskpool.Task(histogram, pixelBuffer);
  let result = await taskpool.execute(task, taskpool.Priority.HIGH);
}
```

注意两个约束：`@Concurrent` 函数里捕获的外部变量会被序列化带过去（失败日志 `taskpool: failed to serialize arguments`，可序列化类型只有普通对象、ArrayBuffer、SharedArrayBuffer、Transferable、Sendable 五种）；函数体内不能访问 UI、不能用主线程的 Context。

## Worker：常驻线程

Worker 是另一种形态：**一个独立的、长期存活的 ArkTS 线程**，有自己的事件循环，通过 `postMessage` / `onmessage` 与主线程互发消息。它适合"有一个长期存在的工作者"的场景：持续解析数据流、维护一个常驻的计算引擎、游戏逻辑线程。

```typescript
// 主线程
let w = new worker.ThreadWorker('entry/ets/workers/DataWorker.ets');
w.onmessage = (e) => { /* 收结果 */ };
w.postMessage({ cmd: 'start', payload: config });
```

Worker 的创建和销毁开销比提交一个 TaskPool 任务大得多（要初始化一个完整 ArkTS 环境），官方建议不要频繁创建销毁。另外 Worker 数量有限制——官方并发设计文档给出的上限是 **64 个**，超了创建会失败。

## 官方对比数据与选型规则

官方《TaskPool 和 Worker 的对比》文档给过一组中载模型实验数据，照原文转述：并发任务数为 1 时，TaskPool 与 Worker 执行完任务用时相近；随着并发任务数增多，TaskPool 逐渐优于 Worker——因为它支持优先级，系统资源不足时高优先级任务更容易拿到资源；任务数为 4 时出现了一个有趣的反转，TaskPool 比 Worker 耗时略长，单任务场景下 Worker 带来的性能收益更大。内存维度上，TaskPool 在 Worker 之上多实现了调度器和线程池，任务多时多占内存——**内存受限设备或任务量大的场景，Worker 的运行时内存占用更低**。

把官方建议归纳成选型规则：

| 场景 | 推荐 |
| --- | --- |
| 一批相互独立的耗时任务（图像处理、批量计算），时长不超 3 分钟同步执行 | TaskPool |
| 需要设置优先级、取消、任务组编排 | TaskPool |
| 常驻运行的工作者（长连接、持续数据处理、超 3 分钟的长任务） | Worker |
| 任务间有关联、需要按序协作的同步任务 | Worker（官方同步任务文档：任务相对独立用 TaskPool，有关联用 Worker） |
| 对内存占用敏感、任务量巨大 | Worker（运行时占用更低） |
| 高频次、单次极短的任务 | 都不用——官方 TaskPool 使用指导：耗时十分短的任务直接放主线程，并发本身有执行成本 |

还有一条铁律写在并发注意事项里：**避免在并发线程中操作 UI**。UI 操作必须在主线程执行，并发线程里操作 UI 可能导致界面异常或崩溃。拿到计算结果后要回主线程更新状态。

## 在日志与 Trace 中看 TaskPool

TaskPool 的调度活动有专门的 hilog（官方并发 FAQ 给了格式）：

```
taskpool:: Task Allocation: taskId, priority        // 任务分配
taskpool:: maxThreads: maxNum, created num: createNum, total num: totalNum  // 扩容
taskpool:: Task Perform: name, taskId, runningLoop: loopId                  // 执行
```

`hdc shell "hilog | grep taskpool"` 就能看到任务提交、线程扩容、任务执行三条线。怀疑"任务没跑"时先看 Allocation 有没有打；怀疑"线程不够"看 maxThreads 那行的 create/total 是否到顶。

在 DevEco Profiler 的 CPU 会话里，TaskPool 工作线程是独立线程泳道，耗时任务在哪个线程跑了多久一目了然。主线程卡顿排查时，先确认卡的那段时间主线程在执行什么——如果该挪走的重计算还在主线程泳道里，答案已经摆在眼前。

## 版本演进

- API 9–11：TaskPool 与 Worker 随 Stage 模型提供，五种可序列化类型定型；
- API 12（5.0）：@Sendable 共享对象、AsyncLock 等补进并发体系（5.2 节）；官方发布 taskpool-vs-worker 对比文档与数据；
- API 20+（6.0）：TaskGroup 支持超时与优先级配置（execute(group, configs)）；长时任务场景给出 TaskPool + emitter 的标准做法；线程数上限、扩缩容日志格式在并发 FAQ 中公开。

## 常见问题与误区

**"TaskPool 就是线程池，想用几个线程用几个"。** 线程数由系统按物理核数和负载管理，开发者控制不了也不该控制。需要固定一个线程跑关联任务时，用 Worker 而不是想办法"固定"TaskPool 线程。

**"序列化失败是框架 bug"。** 几乎总是传了不可序列化的东西（函数、Promise、宿主线程的 Context、未标记 @Sendable 的 class 实例）。按错误日志定位到具体参数。

**"并发开了性能就一定好"。** 并发有序列化成本、调度成本、线程切换成本。任务太碎时总耗时反超主线程直跑，官方"耗时十分短的任务放主线程"说的就是这个。

## 参考资料

- 华为开发者文档：《TaskPool 和 Worker 的对比》（taskpool-vs-worker：中载模型实验数据、内存占用对比、选型建议）
- 华为开发者文档：《TaskPool 简介》（taskpool-introduction：默认单线程、自动扩容、上限为物理核数、缩容）
- 华为开发者文档：《多线程并发概述》（multi-thread-concurrency-overview：Actor 模型、避免并发线程操作 UI）
- 华为开发者文档：《TaskPool 使用指导》（task-pool-usage-guidelines：五种可序列化类型、失败日志、短任务放主线程、TaskGroup）
- 华为开发者文档：《CPU 密集型任务开发》（cpu-intensive-task-development：3 分钟界限）、《同步任务开发》（sync-task-development：独立用 TaskPool、关联用 Worker）
- 华为开发者文档：《并发 FAQ》（concurrency-faq：Task Allocation/maxThreads/Task Perform 日志格式）
- 华为开发者文档：《应用并发设计》（bpta-app-concurrency-design：Worker 限制 64 个）
- OpenHarmony 仓库 `openharmony/arkcompiler_ets_runtime`（TaskPool 调度器与 Worker 实现）
