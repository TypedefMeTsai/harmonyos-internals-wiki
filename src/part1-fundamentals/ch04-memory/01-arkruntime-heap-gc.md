# 4.1 方舟运行时内存模型与 GC

## 为什么要懂运行时的堆

ArkTS 开发者不写 `free`，不等于内存问题不存在。方舟运行时的 GC 替我们回收垃圾，但它回收不了"还被引用着"的对象——泄漏在托管语言里照样发生，只是形态从"忘了释放"变成"忘了断开引用"。而 GC 自己也有代价：标记、拷贝、整理都要花时间，分配越快、存活对象越多，GC 越频繁，主线程上出现的卡顿就有 GC 的一份。

懂运行时的内存模型，才能回答两个实际问题：Snapshot 里那一堆对象为什么还活着？GC 日志里那次几十毫秒的停顿是什么触发的？

## ArkCompiler：从源码到运行时

先建立背景。方舟编译运行时（ArkCompiler）分两半：**ArkTS 编译工具链**把 ArkTS/TS/JS 源码编译成方舟字节码（.abc 文件，发布态打进 HAP），**ArkTS 运行时**在设备上执行字节码。运行时包含解释器、JIT/AOT 优化和内存管理子系统，OpenHarmony 仓库对应 `arkcompiler/ets_runtime`（运行时）与 `arkcompiler/ets_frontend`（es2abc 编译器）。

每个 ArkTS 引擎实例（一个应用进程通常一个，见 1.3 节）有自己独立的堆。Worker/TaskPool 线程有各自独立的虚拟机环境和堆——这正是 Actor 模型下对象默认要序列化传递的原因（5.1 节），也是 Sendable 共享堆设计（5.2 节）要解决的矛盾。

## HPP GC：分代模型与混合算法

方舟运行时的垃圾回收器叫 **HPP GC（High Performance Partial Garbage Collection，高性能部分垃圾回收）**。官方 GC 介绍文档把"High Performance"拆成三个方面：分代模型、混合算法和 GC 流程优化。

**分代模型**基于弱分代假说：大多数新分配的对象在第一次 GC 后就会被回收，而熬过多次 GC 的对象大概率长期存活。运行时将对象划分为年轻代和老年代，分配到不同空间：

- **年轻代**：新对象的出生地。回收频繁、单次快，用拷贝式算法（存活对象少，拷贝成本低），叫 Partial GC / Young GC 一类；
- **老年代**：熬过若干次年轻代回收的对象晋升到这里。回收频率低、单次贵，用标记-清除/标记-整理类算法；
- **大对象与特殊空间**：超过阈值的大对象、JIT 代码、对象布局信息各有专属空间[待验证：ArkTS 运行时各空间的确切命名与容量策略，不同版本有调整]。

**混合算法**指不同区域用不同回收策略的组合，加上并发标记等手段把 STW（Stop-The-World）时间切碎。官方没有公开完整的停顿时间数据，我们不应编造数字；实际停顿以真机 GC 日志和 Profiler 为准。

**回收的可达性判据**是标记-清除模型：运行时维护一棵以 GC Root 为根的引用树，从根出发可达的对象标记为存活，不可达的回收。官方内存分析文档特别提醒过一个误解：GC Root（Snapshot 中 distance 为 1 的节点）不是"要回收的"，恰恰相反——挂在 GC Root 上的对象是肯定存活的，GC 每次都是从根开始遍历。分析泄漏时，我们看的是"这个本该死的对象被哪条引用链挂到了根上"。

## 对象分配与回收的实际含义

把机制翻译成工程直觉：

- **临时对象是最便宜的。** 循环里 new 的小对象，熬不过下一次年轻代回收，回收几乎零成本。真正贵的是**中途存活下来的对象**——它们经历拷贝、晋升，每次 GC 都被反复搬运。所以性能优化里"减少热路径上的中间态大对象"比"一个都不许 new"更准确。
- **大对象直接进专属空间**，分配和回收都走单独路径。频繁创建销毁大对象（比如每帧 new 一个大数组）是 GC 压力的放大器，该复用 buffer 就复用（图片场景用 PixelMap 池、并发场景用 SharedArrayBuffer）。
- **引用不断，永不回收。** 常见挂法：全局单例持有了页面组件、eventHub/emitter 订阅没注销、定时器回调闭包捕获了 this、缓存 Map 只进不出。Snapshot 分析的全部工作就是找到这条链。
- **跨线程的引用规则不同。** Sendable 对象在并发实例间共享，回收要考虑多堆引用（5.2 节），不能用普通对象的直觉推断生命周期。

## 在工具中看 GC 与堆

**Snapshot（堆快照）**：DevEco Profiler 的 Snapshot 会话在任意时刻 dump 堆，按类聚合对象，看 retained size 和到 GC Root 的引用链（distance 字段）。定位泄漏的标准动作：操作前后各抓一份，对比增量对象，挑"该随页面销毁却没销毁"的类，沿引用链找持有者。第 21 章有完整案例。

**Allocation（分配采样）**：看一段时间内的分配热点——哪个函数在大量分配什么类型。滑动卡顿且 GC 频繁时，先在列表项 build 路径上找分配热点。

**日志侧**：方舟运行时的 GC 活动会打 hilog（过滤 ArkCompiler 相关 tag）。并发序列化失败这类问题也在这里留痕——官方 TaskPool 文档提到序列化失败日志 `taskpool: failed to serialize arguments` 可以过滤 ArkCompiler Error 日志查看。

**系统视角**：`hdc shell "hidumper --mem <pid>"` 看进程整体内存构成（ArkTS 堆、Native 堆、图形 buffer 等分区）[待验证：hidumper 内存输出的分区名在各版本的差异]。堆不大但 RSS 很大时，问题多半在 Native 或图形侧，别在 Snapshot 里空找。

## 常见问题与误区

**"托管语言不会内存泄漏"。** 泄漏的定义是"对象存活时间超过需要"，引用不断 GC 就救不了。鸿蒙应用最高发的泄漏形态是订阅未注销和缓存无界。

**"GC 卡顿只能靠换机型解决"。** 先降分配率和存活集：热路径少分配、大对象复用、长列表用复用组件（@Reusable 也减少组件对象 churn）。多数 GC 压力是应用自己造的。

**把 V8 的知识直接套过来。** ArkWeb 内嵌的 Web 引擎用 V8（官方 ArkWeb OOM 文档里老生代细分 Old Pointer Space、Old Data Space、Large Object Space、Code Space、Map Space），那是另一套堆。ArkTS 堆和 Web 堆在同一进程里并存，分析时先分清对象属于哪边。

## 版本演进

- API 9–11：HPP GC 随 Stage 模型服役，分代 + 部分回收框架成形；
- API 12（5.0）：单框架后运行时持续调优，Snapshot/Allocation 工具链在 DevEco Profiler 中成熟；
- API 20+（6.0）：并发能力增强（Sendable 共享堆、AsyncLock 等）对 GC 提出跨堆引用管理的新要求；官方 GC 介绍文档公开分代模型与混合算法的设计；多线程安全检测（setMultithreadingDetectionEnabled）等诊断能力补齐。

## 参考资料

- 华为开发者文档：《GC 介绍》（gc-introduction：HPP GC、分代模型、混合算法与流程优化）
- 华为开发者文档：《方舟运行时与编译工具链概述》（guidebook solution2：.abc 字节码、es2abc、运行时职责）
- 华为开发者文档：《ArkTS/JS 内存分析》（bpta-arkts-js-memory-analysis：GC Root 与 distance=1、标记-清除可达性）
- 华为开发者文档：《ArkWeb OOM 故障模式》（bpta-overview-of-arkweb-oom-fault-modes：V8 老生代空间细分，对比用）
- 华为开发者文档：《TaskPool 使用指导》（task-pool-usage-guidelines：序列化失败日志）
- OpenHarmony 仓库 `openharmony/arkcompiler_ets_runtime`（HPP GC、堆管理）、`openharmony/arkcompiler_ets_frontend`（es2abc）
