# 13.2 Frame 模板与关键 Trace 点速读

## 为什么需要一张速读表

打开一份 Frame 相关的 Trace，泳道几十条、打点几千个。新手的问题不是信息不够，是信息太多——不知道哪个名字对应渲染管线的哪一环，看到一个长切片不知道它是正常的还是异常的。

本节把渲染链路上最高频的 Trace 点整理成速读表。组织方式沿数据流向：Vsync 信号 → 应用主线程处理 → 提交 RS → RS 绘制合成 → 上屏，外加几条支线（动画、懒加载、自渲染、Web）。每个 Trace 点按三问回答：**正常什么样、异常什么样、指向什么问题**。这张表与第 7、8 章的案例互为索引——案例里出现的每个判断，都能在这里找到依据。

判读的通用基准先记住：帧预算由刷新率决定（60Hz 16.7ms / 90Hz 11.1ms / 120Hz 8.3ms），任何"每帧都该出现"的 Trace 点，单次耗时接近或超过帧预算的一半就值得警惕。

## 主线一：Vsync 信号

**`H:ReceiveVsync`**（应用主线程 / RS RenderThread）
- 正常：以屏幕刷新率为间隔规律出现，间隔均匀。
- 异常：间隔忽大忽小（Vsync 被抢占或频率切换），或者该来的时候没来。
- 指向：渲染循环的节拍器，一切帧分析的对齐基准。应用在它之后开始一帧的工作。

**`H:OnVsyncEvent`**（应用线程）
- 正常：紧跟 ReceiveVsync，耗时很短。
- 异常：本身耗时拉长，或与 ReceiveVsync 之间有明显间隔。
- 指向：官方定义"收到 Vsync 信号，渲染流程开始"（出处：《点击完成时延分析》）。它和 ReceiveVsync 之间的间隔反映主线程当时是否被占用——Vsync 到了但主线程在忙别的，事件只能排队。

**`H:SendVsyncTo conn: xxx` / `H:xxx_RequestNextVsync` / `H:ReceiveVsync name:xxx`**（VsyncGenerator / RS OS_IPC / 各申请者）
- 正常：名单里的申请者都是有实际绘制需求的，静止页面时名单收敛。
- 异常：静止页面上仍有申请者持续以 60Hz 请求（如遗忘的 displaySync、框架帧回调）。
- 指向：Vsync 空刷与功耗问题，详见第 11.2 节（出处：《Vsync 低功耗优化》）。

## 主线二：应用主线程帧处理

这一段在 Trace 上嵌套为 `FlushVsync` 内的几个子阶段，按执行顺序读：

**`H:FlushVsync`**
- 正常：包含完整的帧处理过程，总时长在帧预算内。
- 异常：总时长超预算。
- 指向：官方定义"刷新视图同步事件，包括记录帧信息、刷新任务、绘制上下文、处理用户输入"。它是应用侧一帧工作的总入口，超了再往下查子阶段。

**`H:FlushDirtyNodeUpdate`**
- 正常：耗时与被标脏的组件数量成正比，常规刷新下很短。
- 异常：耗时拉长，说明本次刷新波及的组件太多。
- 指向：标脏范围过大。状态变量设计不合理（粗粒度 @State、@Prop 深拷贝、@Observed 嵌套对象整树刷新）是首因。官方建议：页面刷新时尽量减少刷新的组件数量（出处：《点击完成时延分析》）。

**`JSView:ExecuteRerender`**
- 正常：仅在被标脏的自定义组件上出现。
- 异常：一帧内大量组件都在 Rerender，包括视觉上看不出变化的组件。
- 指向：冗余重建。配合 ArkUI Component 泳道看是哪些组件，回查状态绑定范围。

**`H:FlushLayoutTask`**
- 正常：耗时与标脏组件的布局复杂度成正比。
- 异常：单次耗时数毫秒甚至更高，展开后 Measure 占大头。
- 指向：布局嵌套过深或参与布局的组件太多。官方案例：20 层 Stack 嵌套的列表项导致 FlushLayoutTask 超时（第 7.2 节）。用 ArkUI Inspector 看组件树确认。

**`Builder:BuildLazyItem [index]`**
- 正常：滑动中按需出现，单个耗时亚毫秒级；开启组件复用后变为 BuildRecycle，更短。
- 异常：每个滑动帧都出现且单次耗时数毫秒。
- 指向：列表项创建成本过高。官方数据：组件复用前 10.277ms，复用后 0.749ms（出处：《ArkUI 布局优化指导》）。

**`H:FlushRenderTask`**
- 正常：包含实际的绘制指令生成，耗时稳定。
- 异常：耗时拉长；或另一极端——该有的时候是空的。
- 指向：前者查绘制内容复杂度；后者（空 FlushRenderTask 但帧还在跑）是 Vsync 空刷的典型特征，见第 11.2 节。

**`H:FlushMessages` → `H:SendCommands` → `H:MarshRSTransactionData`**
- 正常：三个点依次出现且耗时短，MarshRSTransactionData 有数据送出。
- 异常：FlushMessages 与 SendCommands 之间间隔大；或 MarshRSTransactionData 缺失。
- 指向：这是应用把绘制指令提交给 RS 的三连点，本身耗时短到需要搜索定位（出处：《Launch 分析案例》）。MarshRSTransactionData 缺失说明本帧没有内容提交 RS——空刷嫌疑。

## 主线三：RS 侧绘制合成

**`H:ReceiveVsync`**（render_service 的 RenderThread）
- 正常：与屏幕刷新率对齐，随后出现 ProcessCommandUni。
- 指向：RS 侧一帧的起点。

**`H:ProcessCommandUni`**
- 正常：处理应用提交来的指令，耗时与指令量成正比。
- 异常：耗时拉长。
- 指向：RS 侧处理负载过重——界面结构复杂、图层过多。它是首帧响应时延渲染阶段的起点（出处：《点击响应时延分析》）。

**`H:RSMainThread::DoComposition`**
- 正常：合成阶段，耗时与图层数量、合成方式（GPU/HWC）相关。
- 异常：耗时持续偏高。
- 指向：图层数量过多或合成路径低效，典型于大量重叠半透明、模糊效果。RenderDeadlineMissed 的主嫌疑区。

**`H:RSHardwareThread::CommitAndReleaseLayers`**
- 正常：每帧末尾出现，耗时短。
- 指向：送显提交点，首帧响应时延渲染阶段的终点（出处：《点击响应时延分析》）。

**`H:Waiting for PresentFence` / PresentFence 泳道**
- 正常：每帧都有一段等待 PresentFence，长度适中。
- 异常：泳道上出现空白缺口。
- 指向：官方 FAQ：滑动范围内 PresentFence 泳道缺少 `H:Waiting for PresentFence` 的部分，很可能存在丢帧，需结合应用侧和 RS 侧主线程 Trace 确认（出处：《性能 FAQ-53》）。

## 支线：动画、自渲染与 Web

**`H:Animator`**
- 正常：动画时长与设定一致，帧率贴合期望帧率。
- 异常：动画周期内频繁红帧，或动画时长明显超出设计值。
- 指向：转场/动效耗时。官方对照实验：animationDuration 100ms vs 1000ms，完成时延 99ms39μs vs 1s7ms693μs（出处：《点击完成时延分析》）。自定义动画曲线计算在主线程跑也是这里暴露。

**`H:DispatchDisplaySync`（含 DisplaySyncId、DrawFPS/VsyncRate 信息）**
- 正常：只在组件可见且有内容更新时出现。
- 异常：组件不可见后仍周期性出现。
- 指向：displaySync 生命周期管理缺失，空刷功耗问题（第 11.2 节）。

**`H:RosenWeb`（Web 场景）**
- 正常：RosenWeb 泳道的 buffer 计数大于 0。
- 异常：VSyncGenerator 发 RS 信号时 buffer 个数为 0。
- 指向：Web 为自渲染模式，丢帧判定看 buffer 缓存状态：buffer 为 0 即丢帧；往前推几个 Vsync 周期找原因，Web 滑动丢帧通常归为执行超时或缺少必要渲染条件（出处：《Web 帧率性能分析》）。

**`H:DispatchTouchEvent`**
- 正常：按下 type=0、离手 type=1 成对出现，与后续帧处理衔接紧凑。
- 异常：事件到达与主线程开始处理之间有明显间隔。
- 指向：响应/完成时延分析的起点标志；间隔大说明主线程被占。

## 使用这张表的方法

分析任何 Frame Trace 时按固定顺序走：先在 Frame 泳道定界（App 红还是 RS 红，7.1 节）→ 红在 App，沿主线二从上往下找第一个异常拉长的阶段 → 红在 RS，查主线三的 ProcessCommandUni 和 DoComposition → 时延问题从 DispatchTouchEvent 出发沿主线走全程 → 功耗问题查支线的 Vsync 申请关系。表里没有答案时，答案通常在"两个 Trace 点之间的空白"里——空白意味着等待，用 CPU 调度泳道的唤醒链（WakeUP From Tid）找它在等谁，方法见第 8.2 节。

## 参考资料

- 《帧率问题分析》（bpta-zhenlv）
- 《点击完成时延分析》（bpta-click-to-complete-delay-analysis）
- 《点击响应时延分析》（bpta-click-to-click-response-optimization）
- 《Vsync 低功耗优化》（bpta-vsync-power-optimization）
- 《Frame 分析》（ide-insight-session-frame）
- 《Web 帧率性能分析》（bpta-web-frame-rate-performance-analysis）
- 《性能 FAQ》（faqs-performance-53）
- 《Launch 分析案例》（ide-profiler-launch-case）
