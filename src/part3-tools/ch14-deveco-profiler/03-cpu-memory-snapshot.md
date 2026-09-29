# 14.3 CPU / Memory / Snapshot 会话

## CPU 会话：采样、调用栈与唤醒链

CPU 会话回答的问题：这段时间里 CPU 在跑什么、谁在抢 CPU、线程在干活还是在等。

**泳道结构**：进程泳道显示进程对各 CPU 核的占用，展开后是线程列表及线程运行状态（Running/Runnable/Sleeping 等）；CPU Core 泳道给出每个核的频率与负载。工具栏可切换**精简模式**——只采集当前进程、大桌面进程和 render_service 进程的 trace，数据量大幅减少，长场景录制建议开（出处：《CPU 活动分析》）。

**两种分析视图**：

- **采样视图**：按固定频率采集各线程的调用栈，聚合成火焰图式的热点视图。找"CPU 时间花在哪个函数"用它，右侧按权重排序，权重最高的调用栈就是头号消耗者。
- **Trace 视图**：线程泳道上的切片（系统 Trace 线程名为"线程名+线程号"，用户 HiTraceMeter 打点显示为打点任务名），点选切片看名称、进程、线程、起始时间、时长、深度。

**View Integrated Scheduling Chain（唤醒链）**：开启后，点击 CPU 时间片泳道的节点，可以查看某个 CPU 上运行线程的完整唤醒调度链——谁唤醒了它、它又唤醒了谁（出处：《CPU 活动分析》）。这是第 8.2 节"主线程空闲 397ms 去哪了"案例的工具化答案：没有它要靠 Runnable 详情里的 WakeUP From Tid 一段段手跳，有它一条链直接展开。主线程莫名等待、线程间互相踢皮球这类问题，先开这个开关。

**与 Frame 会话的联动**：CPU 泳道在 Frame 模板里同样存在。丢帧分析的第一步"排除系统侧"（线程状态、运行核、频率，见 7.1 节）就是在 CPU Core / Process 泳道上完成的。看到一个红帧，先看对应时刻主线程是不是 Running、跑在哪个核上，再去看 Callstack。

## Memory 会话：曲线先行

Memory 会话持续采样进程的内存指标，输出时间曲线。它的定位是**发现与定界**，不是定位——曲线能告诉你内存在涨、什么时候涨、涨得多快，但不知道是什么对象。

标准用法（出处：《ArkTS 内存泄漏场景化最佳实践》）：录制期间重复执行可疑操作（如"进入页面 → 停留 2 秒 → 返回"× N 次），观察曲线形态：

- 每次退出后回落至基线附近，整体呈**锯齿状**——正常；
- 阶梯状上升、无明显回落——泄漏嫌疑（官方案例中多次操作后涨到 314.1MB 不回落，判定泄漏）。

确认泄漏嫌疑后，不要继续在 Memory 会话里耗，转 Snapshot。

## Snapshot 会话：堆快照与对比

Snapshot 拍摄 ArkTS 堆的完整快照。核心机制先理解：**每次拍摄前虚拟机会触发 GC**，所以快照里存在的对象理论上都是 GC 已经无法回收的（出处：《ArkTS 内存泄漏分析》）。这天然筛掉了临时对象，留下的都是"有理由活着"的对象——泄漏分析就是在这些"理由"里找不合法的那个。

**操作路径**：Profiler → 选进程 → Snapshot 模板 → 创建 Session → 录制。在可疑操作前拍第一份（Snapshot1），重复操作后拍第二份（Snapshot2），必要时拍第三份。

**三个分析视图**（出处：《Snapshot 模板基本操作》《JSVM 内存分析》）：

- **Comparison（对比）**：选一份为 base、另一份为 Target，输出两份快照的差异：新增数、删除数、个数增量、分配大小（Alloc Size）、释放大小（Freed Size）、大小增量（Size Delta）。泄漏排查的主视图——按 Size Delta 排序，"新增且不释放"的对象类型排前面。
- **Containment（引用树）**：自上而下的树形界面，浏览任意对象的引用情况。从 Comparison 里的嫌疑对象跳过来，沿引用链向上找持有者：谁在引用它 → 谁又在引用持有者 → 一直追到 GC Root。泄漏的答案通常是链条上某个忘了断开的环节（未反注册的监听器、只增不减的静态集合）。
- **Statistics（统计）**：饼图展示各类型对象的内存占比，用于快速建立"堆里都是什么"的直觉。

**离线导入**：Profiler 会话区 Open File 可导入一个或多个 `.heapsnapshot` 文件（出处：《JS 内存泄漏检测》）。线上或同事抓的现场可以直接拿来看。Native 侧可用 `OH_JSVM_TakeHeapSnapshot` 主动拍快照，注意它会短暂暂停应用运行，回调在 VM 线程同步执行、不要做阻塞操作，频繁调用会产生大量文件（出处：《JSVM 堆内存管理》）。

## Allocation 会话：带调用栈的泄漏检测

Snapshot 对比能找到"什么对象漏了"，但找到"在哪段代码分配的"要绕引用链。Allocation 模板的 **Handle 泄漏检测**更直接：录制泄漏场景，工具采集分配信息，直接给出泄漏对象对应的调用栈，精准定位到代码中的调用关系（出处：《ArkTS 内存泄漏总览》）。

配置步骤：Profiler 创建 Allocation 模板 → 在过滤泳道中添加 ArkTS Snapshot 泳道 → 在录制配置中开启泄漏检测 → 录制泄漏场景。采集 native 堆分配调用栈的 native hook 插件只支持调试证书签名的应用（出处：《hiprofiler》）。

实务建议：Memory 曲线定性 → Snapshot 对比定对象 → Allocation 定代码位置，三步各司其职，不要试图用一个会话解决全部问题。

## ArkWeb 分析：Sub Resource

嵌入 WebView 的应用有专属的 ArkWeb 分析模板。它结合 ArkWeb 执行流程的关键 trace 点定位问题阶段：问题发生在渲染阶段时，结合 `H:RosenWeb` 数据、线程运行状态和帧渲染流程打点进一步分析丢帧（出处：《ArkWeb 分析》）。Sub Resource 相关泳道展示页面子资源（JS/CSS/图片/字体）的加载瀑布，哪个资源阻塞了首屏一目了然。Web 丢帧的判定逻辑比较特殊——看 RosenWeb 的 buffer 缓存状态而非普通帧泳道，VSyncGenerator 发信号时 buffer 为 0 才算丢帧（出处：《Web 帧率性能分析》），判读细节见第 13.2 节支线部分。

## 常见问题与坑

**Snapshot 拍出来的对象名字看不懂**。release 混淆后类名方法名被压缩，需配合对应版本的 Source Map 解析；开发期用 debug 包拍，名字直接可读。

**两次快照之间对象全部"新增"**。操作之间触发了页面重建或数据源重载，旧对象树整体被替换——这不是泄漏，是对比基准选错了。对比应在"回到同一界面状态"的两个时刻之间做。

**Allocation 录制后应用明显变卡**。分配追踪是重开销采集，只用于定位阶段，不要拿它测性能指标，也不要长时间录制。

**CPU 采样热点指向 program**。program 表示纯 Native 执行段，无 ArkTS 栈信息，切到 Callstack（Native）泳道看；那里也看不出有效信息时，热点通常在系统库或引擎内部，考虑从调用入口（ ArkTS 侧）减少调用次数。

## 参考资料

- 《CPU 活动分析》（ide-insight-session-cpu）
- 《Snapshot 模板基本操作》（ide-snapshot-basic-operations）
- 《ArkTS 内存泄漏分析》（ide-arkts-memory-leak-analysis）
- 《ArkTS 内存泄漏总览》（bpta-overview-of-arkts-memory-leaks-overview）
- 《ArkTS 内存泄漏场景化最佳实践》（bpta-arkts-leak-in-develop）
- 《JS 内存泄漏检测》（bpta-stability-js-memleak-detection）
- 《ArkWeb 分析》（ide-profiler-arkweb）
- 《Web 帧率性能分析》（bpta-web-frame-rate-performance-analysis）
- 《hiprofiler》（hiprofiler）
