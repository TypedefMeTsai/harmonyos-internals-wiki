# 目录

- [写在前面](preface/intro.md)
- [许可说明](preface/license.md)
- [本书的使用方式](preface/how-to-use.md)
- [适用读者](preface/target-audience.md)
- [版本约定](preface/version-conventions.md)
- [阅读路径推荐](preface/reading-paths.md)

---

# 第一部分：HarmonyOS 系统运行机制

- [第 1 章：系统架构全景](part1-fundamentals/ch01-architecture/README.md)
  - [1.1 四层架构与内核可替换设计](part1-fundamentals/ch01-architecture/01-four-layer-architecture.md)
  - [1.2 OpenHarmony 与 HarmonyOS 的关系](part1-fundamentals/ch01-architecture/02-openharmony-and-harmonyos.md)
  - [1.3 Stage 模型、Ability 与进程模型](part1-fundamentals/ch01-architecture/03-stage-model-ability-process.md)
  - [1.4 系统服务框架：SAMgr、init 与进程骨架](part1-fundamentals/ch01-architecture/04-samgr-init-process-skeleton.md)
  - [1.5 分布式软总线与超级终端](part1-fundamentals/ch01-architecture/05-dsoftbus-super-device.md)

- [第 2 章：ArkUI 渲染系统](part1-fundamentals/ch02-rendering/README.md)
  - [2.1 渲染架构总览：从 ArkTS 声明到像素](part1-fundamentals/ch02-rendering/01-rendering-architecture.md)
  - [2.2 Vsync、统一渲染与一帧的生命周期](part1-fundamentals/ch02-rendering/02-vsync-unified-rendering.md)
  - [2.3 状态管理、脏区刷新与组件重建](part1-fundamentals/ch02-rendering/03-state-dirty-rebuild.md)
  - [2.4 Render Service、RSNode 与 HWC 送显](part1-fundamentals/ch02-rendering/04-render-service-hwc.md)
  - [2.5 自渲染管线：XComponent、NativeWindow 与 NativeVSync](part1-fundamentals/ch02-rendering/05-self-rendering-pipeline.md)

- [第 3 章：输入系统](part1-fundamentals/ch03-input/README.md)
  - [3.1 多模输入框架与事件分发](part1-fundamentals/ch03-input/01-multimodal-input-dispatch.md)
  - [3.2 触摸响应链与跟手性](part1-fundamentals/ch03-input/02-touch-response-chain.md)
  - [3.3 手势识别与系统手势导航](part1-fundamentals/ch03-input/03-gesture-recognition.md)

- [第 4 章：内存管理](part1-fundamentals/ch04-memory/README.md)
  - [4.1 方舟运行时内存模型与 GC](part1-fundamentals/ch04-memory/01-arkruntime-heap-gc.md)
  - [4.2 系统内存压力治理与进程回收](part1-fundamentals/ch04-memory/02-memory-pressure-reclaim.md)

- [第 5 章：并发与调度](part1-fundamentals/ch05-concurrency-scheduling/README.md)
  - [5.1 ArkTS 并发模型：Actor、TaskPool 与 Worker](part1-fundamentals/ch05-concurrency-scheduling/01-actor-taskpool-worker.md)
  - [5.2 Sendable 共享对象与线程间通信](part1-fundamentals/ch05-concurrency-scheduling/02-sendable-shared-objects.md)
  - [5.3 FFRT、异步 I/O 与系统任务调度](part1-fundamentals/ch05-concurrency-scheduling/03-ffrt-async-io-scheduling.md)

- [第 6 章：存储与 I/O](part1-fundamentals/ch06-storage/README.md)
  - [6.1 应用沙箱与文件系统视图](part1-fundamentals/ch06-storage/01-sandbox-filesystem.md)
  - [6.2 Preferences、KV Store 与关系型数据库](part1-fundamentals/ch06-storage/02-preferences-kv-rdb.md)

---

# 第二部分：性能问题与优化

- [第 7 章：流畅性](part2-performance/ch07-smoothness/README.md)
  - [7.1 丢帧故障模型：AppDeadlineMissed 与 RenderDeadlineMissed](part2-performance/ch07-smoothness/01-deadline-missed-models.md)
  - [7.2 滑动丢帧分析实战：长列表案例](part2-performance/ch07-smoothness/02-list-jank-case.md)

- [第 8 章：响应速度](part2-performance/ch08-responsiveness/README.md)
  - [8.1 点击响应链路与完成时延](part2-performance/ch08-responsiveness/01-click-response-chain.md)
  - [8.2 转场时延分析与优化](part2-performance/ch08-responsiveness/02-transition-latency.md)

- [第 9 章：应用无响应（AppFreeze）](part2-performance/ch09-appfreeze/README.md)
  - [9.1 AppFreeze 判定机制与故障类型](part2-performance/ch09-appfreeze/01-appfreeze-mechanism.md)
  - [9.2 AppFreeze 日志分析与定位](part2-performance/ch09-appfreeze/02-appfreeze-log-analysis.md)

- [第 10 章：内存性能](part2-performance/ch10-memory-perf/README.md)
  - [10.1 内存性能问题分类与排查路径](part2-performance/ch10-memory-perf/01-memory-issue-taxonomy.md)

- [第 11 章：功耗](part2-performance/ch11-power/README.md)
  - [11.1 功耗模型与后台资源管控](part2-performance/ch11-power/01-power-model-background.md)
  - [11.2 长时任务与唤醒治理](part2-performance/ch11-power/02-long-task-wakeup.md)

- [第 12 章：网络性能](part2-performance/ch12-network/README.md)
  - [12.1 网络栈与请求性能](part2-performance/ch12-network/01-network-stack-performance.md)

---

# 第三部分：性能工具与方法论

- [第 13 章：HiTrace 与 SmartPerf](part3-tools/ch13-hitrace-smartperf/README.md)
  - [13.1 HiTrace：bytrace/hitrace 命令与 tag 体系](part3-tools/ch13-hitrace-smartperf/01-hitrace-command-tags.md)
  - [13.2 Frame 模板与关键 Trace 点速读](part3-tools/ch13-hitrace-smartperf/02-frame-template-key-traces.md)
  - [13.3 SmartPerf 与 SmartPerf-Host](part3-tools/ch13-hitrace-smartperf/03-smartperf.md)

- [第 14 章：DevEco Profiler](part3-tools/ch14-deveco-profiler/README.md)
  - [14.1 Frame 会话：丢帧与帧耗时分析](part3-tools/ch14-deveco-profiler/01-frame-session.md)
  - [14.2 Launch 会话：启动性能分析](part3-tools/ch14-deveco-profiler/02-launch-session.md)
  - [14.3 CPU / Memory / Snapshot 会话](part3-tools/ch14-deveco-profiler/03-cpu-memory-snapshot.md)

- [第 15 章：HiLog 与崩溃分析](part3-tools/ch15-hilog-crash/README.md)
  - [15.1 hilog 体系与高效查日志](part3-tools/ch15-hilog-crash/01-hilog-system.md)
  - [15.2 FaultLog、CppCrash 与应用事件](part3-tools/ch15-hilog-crash/02-faultlog-cppcrash.md)

- [第 16 章：方法论](part3-tools/ch16-methodology/README.md)
  - [16.1 性能问题的定义、证据与修复验证](part3-tools/ch16-methodology/01-methodology.md)

---

# 第四部分：系统与设备差异

- [第 17 章：OpenHarmony 系统优化](part4-system/ch17-openharmony/README.md)
  - [17.1 编译构建、镜像裁剪与开机耗时](part4-system/ch17-openharmony/01-build-boot-optimization.md)
- [第 18 章：设备形态与差异](part4-system/ch18-device-form/README.md)
  - [18.1 手机、折叠屏、平板、PC、穿戴与车机](part4-system/ch18-device-form/01-device-forms.md)

---

# 第五部分：应用性能实践

- [第 19 章：启动优化](part5-app/ch19-startup/README.md)
  - [19.1 冷启动链路与阶段拆解](part5-app/ch19-startup/01-cold-start-chain.md)
  - [19.2 启动优化实战清单](part5-app/ch19-startup/02-startup-checklist.md)

- [第 20 章：渲染优化实战](part5-app/ch20-rendering-practice/README.md)
  - [20.1 布局、状态与组件复用](part5-app/ch20-rendering-practice/01-layout-state-reuse.md)
  - [20.2 动画、图片与高阶视效](part5-app/ch20-rendering-practice/02-animation-image-effects.md)
  - [20.3 ArkWeb、视频与 XComponent 场景](part5-app/ch20-rendering-practice/03-arkweb-video-xcomponent.md)

- [第 21 章：内存实践](part5-app/ch21-memory-practice/README.md)
  - [21.1 泄漏、膨胀与 Snapshot 分析实战](part5-app/ch21-memory-practice/01-leak-snapshot-practice.md)

- [第 22 章：I/O 与网络优化](part5-app/ch22-io-network/README.md)
  - [22.1 文件、数据库与序列化优化](part5-app/ch22-io-network/01-file-db-serialization.md)
  - [22.2 请求、缓存与弱网策略](part5-app/ch22-io-network/02-request-cache-weaknet.md)

- [第 23 章：功耗与包体积优化](part5-app/ch23-power-size/README.md)
  - [23.1 应用功耗治理实战](part5-app/ch23-power-size/01-power-practice.md)
  - [23.2 HAP 包体积治理](part5-app/ch23-power-size/02-hap-size.md)

- [第 24 章：应用可观测性](part5-app/ch24-observability/README.md)
  - [24.1 线上质量：崩溃、无响应与性能采集](part5-app/ch24-observability/01-online-quality.md)

---

# 附录

- [附录 A：HarmonyOS 版本与 API 速查](appendix/version-changelog.md)
- [附录 B：常用 hdc / 系统命令](appendix/commands-cheatsheet.md)
- [附录 C：HiTrace tag 与关键 Trace 点速查](appendix/hitrace-cheatsheet.md)
- [附录 D：性能分析 Checklist](appendix/analysis-checklist.md)
- [附录 E：术语表](appendix/glossary.md)
- [附录 F：鸿蒙性能学习路线](appendix/learning-path.md)
