# 21.1 泄漏、膨胀与 Snapshot 分析实战

## 常见泄漏模式：引用从哪来

ArkTS 有 GC，泄漏的含义是"业务上死了、引用上还活着"。高频模式按出现率排序：

**1. 订阅与监听未取消**。`on('xxx', callback)` 注册后没有对应的 `off`。 emitter、eventHub、display 折叠监听、传感器、网络状态监听都是同一个坑。订阅中心持有回调，回调闭包持有组件实例，组件持有整棵子树——页面销毁了，整串对象全活着。

**2. 单例/全局对象持有 context 或组件**。全局工具类、管理器单例缓存了 UIAbilityContext 或某个页面的 this。单例的生命周期等于进程，被它持有的东西永远不释放。

**3. 事件总线挂回调**。全局 EventBus 订阅的事件处理器没注销，和模式 1 同理，只是订阅中心换成了自己的总线实现。

**4. 缓存只进不出**。Map 形式的图片缓存、数据缓存没有容量上限和淘汰策略。这不算严格意义的泄漏（对象"有用"），但效果一样：内存单调上涨直到触发系统管控。系统的资源泄漏检测对内存泄漏（PSS_MEMORY）的判定机制就是：以应用进程平均动态峰值内存作为基线，200s 为基准窗口，动态内存峰值超过基线 2 倍时判定泄漏并触发管控（《资源泄漏检测机制》文档）。

**5. 跨语言引用（Native 侧）**。Native 代码用 `napi_create_reference` 为 ArkTS 对象创建了强引用且一直未删，导致 ArkTS 对象与 Native 对象双重泄漏。这类泄漏在 ArkTS 快照里看不到根，要用 Allocation 工具抓 Native 分配栈，检查 `napi_create_reference` 相关栈帧；DevEco Studio 26.0.0 起，快照导出时可记录 napi_ref 地址及其与 ArkTS 对象的引用关系（需先调 `@util.ArkTSVM.setTrackGlobalRef` 使能），可以直接在快照里按 napi_ref 追。

## Snapshot 快照对比：标准工作流

泄漏定位的核心工具是 DevEco Profiler 的 Snapshot（堆快照），工作流固定五步：

1. **建立基线**：进入应用首页，静置几秒，在 Snapshot 泳道抓第一张快照。
2. **执行可疑操作**：反复执行怀疑泄漏的操作 N 次（比如打开-关闭某个页面 20 次）。
3. **回到基线状态**：操作结束回到首页，静置，抓第二张快照。
4. **对比**：在 ArkTS Snapshot 泳道的 Comparison 区域，选第一张为 base、第二张为 Target，得到比较结果：新增数、删除数、个数增量、分配大小、释放大小、大小增量。
5. **追引用**：按"个数增量"排序，找到 N 次操作后存活数正好多出 N 倍（或接近 N 倍）的类——20 次开关页面后某 ViewModel 类存活 20 个实例，就是它。展开该类实例的引用链（Retainers），从 GC Root 一路往下看：谁持有它 → 谁持有持有者，直到看到订阅中心/单例/全局 Map，定位到代码。

[图：Snapshot Comparison 视图，个数增量排序与单个实例的 Retainers 引用链展开]

离线场景也可以工作：用 `ohos.hiviewdfx.jsLeakWatcher()` 接口在应用内导出 heapsnapshot 或 rawheap 文件，或订阅 OOM 故障事件获取崩溃日志与 rawheap，然后在 Profiler 主界面 "Open File" 离线导入 .heapsnapshot 文件做同样的对比分析。共享堆 OOM 场景还可以调 `hidebug.setProcDumpInSharedOOM()` 开启进程级快照，OOM 发生时自动生成 rawheap。

### 怎么读对比结果而不被骗

- 抓第二张快照前**先强制 GC**（Profiler 提供触发按钮），否则垃圾还没回收，增量里混着"可回收但尚未回收"的对象，虚高。
- 增量为 N 倍才硬气：执行 20 次操作多出约 20 个实例，证据链闭合。多出 1-2 个可能是合理的懒初始化。
- 看 Retained Size 而不只是实例数：100 个小对象可能不如 1 个持大数组的对象严重。ArkWeb 内存分析的最佳实践里，Summary 视图按 Retained size 排序后 97% 内存被一个 bigObject 占用，Containment 视图展开发现是微任务持有的变量——不同视图回答不同问题。

## 内存水位与 GC 观测

Snapshot 回答"谁泄漏"，水位观测回答"离出事还有多远"：

- **Profiler Memory 泳道**：看 PSS 趋势。健康的内存曲线是锯齿形（分配-回收循环）且基线平稳；泄漏的曲线是台阶式上升，每次操作后回落不到原位。
- **onMemoryLevel 回调**：系统内存压力变化会回调应用，分级通知。收到高水位回调时应主动释放可重建资源：图片缓存、预加载数据、复用池。这是应用配合系统内存治理的正式通道，不做就会在更高级别被系统直接处理。
- **GC 观测**：方舟 GC 的停顿与频率可从 Profiler 的 CPU/调用栈数据中观察；频繁 Full GC 且堆持续高位，说明存活对象太多——这时优化方向是减少存活对象（治理缓存与长生命周期对象），而不是调 GC。

## 图片与 PixelMap：最大单项内存的专项治理

多数应用的内存大头是图片。规则在 20.2 讲过机制，这里给内存视角的清单：

- **解码即定尺寸**：用 sourceSize/desiredSize 按显示尺寸解码。一张 12MP 照片按 RGBA_8888 全尺寸解码约 48MB，头像列表里来十张就够触发水位回调。
- **PixelMap 用完即放**：业务侧持有的 PixelMap 在页面销毁、列表项回收时置空释放。图片框架对单张解码有 2GB 上限，但进程限制先到。
- **大图浏览用区域解码 + 下采样组合**：先低清全图，放大时对可视区域 desiredRegion 解码，区域变化后释放不再显示的区域，避免全图 PixelMap 常驻。
- **缓存有上限**：图片缓存按内存大小设上限并做 LRU 淘汰，onMemoryLevel 高水位时主动清空。系统 Image 组件的缓存不可观测，重度图片业务用 ImageKnife 管理缓存策略。

## 膨胀：另一种"内存大"

泄漏是对象死不掉，膨胀是"该用的东西用得太多"。特征：没有泄漏证据（Snapshot 对比干净），但 PSS 基线就是高。常见原因：

- 缓存设计过宽（全量预加载、无淘汰）。
- 重复加载：多 HAP/HSP 引用同一 HAR 时资源多份拷贝（见 23.2）；同一张图多处解码。
- 数据结构低效：大 JSON 全量驻留内存而不是按需解析；该用 Sendable 共享的对象在多线程间各拷一份。

治理思路和泄漏不同：不是找引用链，而是做内存预算——给每个模块定 PSS 配额，Profiler 里按模块归因，超预算的模块自己解释。

## 举一反三：内存问题的排查顺序

拿到一个"内存大"的报告，按这个顺序走：

1. Memory 泳道看曲线形态：台阶上升 → 泄漏方向；高位平稳 → 膨胀方向；尖峰 → 瞬时分配（大图解码、大列表全量创建）。
2. 泄漏方向：Snapshot 对比工作流，个数增量排序 → Retainers 追根。
3. 膨胀方向：模块预算 + 缓存审计。
4. 尖峰方向：Allocation 分析抓瞬时分配栈，对号入座（解码、反序列化、全量加载）。
5. 跨语言嫌疑：Allocation 工具看 napi_create_reference 栈；26.0.0+ 用 napi_ref 快照关联。

## 参考资料

- 官方文档：《Snapshot 基础操作》（ide-snapshot-basic-operations）
- 官方文档：《ArkTS 内存泄漏分析》（ide-arkts-memory-leak-analysis）
- 官方文档：《ArkTS 内存泄漏概述》（bpta-overview-of-arkts-memory-leaks-overview）
- 官方文档：《JS 内存泄漏检测》（bpta-stability-js-memleak-detection）
- 官方文档：《资源泄漏检测机制》（resource-leak-guidelines）
- 官方文档：《N-API 内存泄漏 FAQ》（napi-faq-about-memory-leak）
- 官方文档：《图片 AllocatorType》（image-allocator-type）
- 官方文档：《ArkWeb OOM：JS 对象持有分析》（bpta-arkweb-oom-js-object-held-by-js-object）
