# 鸿蒙技术内幕：系统架构、性能优化与工具实战

> *HarmonyOS Internals: System Architecture, Performance & Tooling in Practice*

面向有经验的 HarmonyOS 开发者和系统工程师。覆盖应用层（ArkTS / ArkUI）、框架层（ArkUI Engine / Ability Framework）、系统服务层（Render Service / Ability Manager / DSoftBus）与内核层（HarmonyOS Kernel / Linux / LiteOS）。当前基准是 HarmonyOS 6.x（API 20+），机制说明对照 OpenHarmony 6.0 开源主干与华为开发者官方文档。

完整目录见 [`src/SUMMARY.md`](src/SUMMARY.md)。

## 怎么读

- 跨层：从应用代码追到 ArkUI Engine、Render Service 和系统服务，同一条问题链写在相邻章节里。
- 按 HarmonyOS / OpenHarmony 版本追踪变化，正文确定性结论最高覆盖 HarmonyOS 6.1 / API 21。
- 机制说明带 OpenHarmony 仓库路径（如 `openharmony/graphic_graphic_2d`），方便回到源码核对。
- 工具章写 HiTrace、DevEco Profiler、SmartPerf 的采集与证据形态；实践章写启动、渲染、内存、I/O、功耗和线上治理。

当前版本为中文。

## 内容结构

### 第一部分：HarmonyOS 系统运行机制

系统怎么把应用跑起来，画面、输入、内存、并发和存储分别在哪一层等待。

- [第 1 章 系统架构全景](src/part1-fundamentals/ch01-architecture/README.md)：四层架构、内核可替换设计、Stage 模型、SAMgr 系统服务、分布式软总线。卡顿、应用无响应或 Native 崩溃时，先确认代码跑在哪个进程、跨过哪条 IPC。
- [第 2 章 ArkUI 渲染系统](src/part1-fundamentals/ch02-rendering/README.md)：从主线程、Vsync、脏区刷新、RSNode 到 Render Service 和 HWC。掉帧、首帧晚，要先分清帧在 App 侧还是 RS 侧超时。
- [第 3 章 输入系统](src/part1-fundamentals/ch03-input/README.md)：多模输入、手势识别与响应链。帧率正常不能证明输入及时到达应用。
- [第 4 章 内存管理](src/part1-fundamentals/ch04-memory/README.md)：方舟运行时 Heap、GC、内存压力治理。OOM 只是其中一种结局。
- [第 5 章 并发与调度](src/part1-fundamentals/ch05-concurrency-scheduling/README.md)：Actor 并发模型、TaskPool、Worker、Sendable、FFRT 与系统调度。
- [第 6 章 存储与 I/O](src/part1-fundamentals/ch06-storage/README.md)：应用沙箱、Preferences、关系型数据库与分布式数据。

### 第二部分：性能问题与优化

- [第 7 章 流畅性](src/part2-performance/ch07-smoothness/README.md)：AppDeadlineMissed 与 RenderDeadlineMissed 两类丢帧故障模型。
- [第 8 章 响应速度](src/part2-performance/ch08-responsiveness/README.md)：从点击到首帧的链路与优化。
- [第 9 章 应用无响应（AppFreeze）](src/part2-performance/ch09-appfreeze/README.md)：THREAD_BLOCK、LIFECYCLE_TIMEOUT 等故障判定与定位。
- [第 10 章 内存性能](src/part2-performance/ch10-memory-perf/README.md)：OOM、泄漏与图形内存。
- [第 11 章 功耗](src/part2-performance/ch11-power/README.md)：后台任务、长时任务与唤醒治理。
- [第 12 章 网络性能](src/part2-performance/ch12-network/README.md)：HTTP/RPC/弱网与网络管理。

### 第三部分：性能工具与方法论

- [第 13 章 HiTrace 与 SmartPerf](src/part3-tools/ch13-hitrace-smartperf/README.md)：hitrace 命令、Frame 模板与关键 Trace 点。
- [第 14 章 DevEco Profiler](src/part3-tools/ch14-deveco-profiler/README.md)：Frame / Launch / CPU / Memory / Snapshot 会话。
- [第 15 章 HiLog 与崩溃分析](src/part3-tools/ch15-hilog-crash/README.md)：hilog、FaultLog、CppCrash 与应用事件打点。
- [第 16 章 方法论](src/part3-tools/ch16-methodology/README.md)：现象定义、证据边界与修复验证。

### 第四部分：系统与设备差异

- [第 17 章 OpenHarmony 系统优化](src/part4-system/ch17-openharmony/README.md)：面向系统开发者的编译、调度与图形栈优化。
- [第 18 章 设备形态与差异](src/part4-system/ch18-device-form/README.md)：手机、折叠屏、平板、PC、穿戴与车机。

### 第五部分：应用性能实践

- [第 19 章 启动优化](src/part5-app/ch19-startup/README.md)
- [第 20 章 渲染优化实战](src/part5-app/ch20-rendering-practice/README.md)
- [第 21 章 内存实践](src/part5-app/ch21-memory-practice/README.md)
- [第 22 章 I/O 与网络优化](src/part5-app/ch22-io-network/README.md)
- [第 23 章 功耗与包体积优化](src/part5-app/ch23-power-size/README.md)
- [第 24 章 应用可观测性](src/part5-app/ch24-observability/README.md)

### 附录

- [附录 A：HarmonyOS 版本与 API 速查](src/appendix/version-changelog.md)
- [附录 B：常用 hdc / 系统命令](src/appendix/commands-cheatsheet.md)
- [附录 C：HiTrace tag 与关键 Trace 点速查](src/appendix/hitrace-cheatsheet.md)
- [附录 D：性能分析 Checklist](src/appendix/analysis-checklist.md)
- [附录 E：术语表](src/appendix/glossary.md)
- [附录 F：鸿蒙性能学习路线](src/appendix/learning-path.md)

## License

正文可公开阅读。引用 OpenHarmony 源码遵循 Apache License 2.0；引用华为开发者官方文档仅用于技术说明目的。
