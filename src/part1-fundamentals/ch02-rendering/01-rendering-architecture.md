# 2.1 渲染架构总览：从 ArkTS 声明到像素

## 为什么要搞清楚这条链路

看一个真实场景：我们在 DevEco Profiler 的 Frame 模板里抓到一帧 AppDeadlineMissed，展开应用进程泳道，看到 `H:FlushMessages` 下面挂着 `H:SendCommands` 和 `H:MarshRSTransactionData`，耗时都不长，但前面的 `UITaskScheduler::FlushTask` 占了 12ms。这组 Trace 点每个词都认识，连起来却不知道该怀疑谁——除非我们已经知道：这些点分布在一条"声明式代码 → FrameNode 树 → 绘制指令 → 跨进程传输 → Render Service → GPU → 屏幕"的流水线上，`FlushTask` 对应流水线中布局与渲染任务的那一段。

本节就把这条流水线画完整。它是后面四节的总地图，也是第 7 章丢帧故障模型的前置知识。

## 链路的五个环节

一次 UI 更新从 ArkTS 代码到像素，经过五个环节：

```
ArkTS 声明（@Component/build） → ① 前端桥接与组件树（应用进程/UI 线程）
→ ② FrameNode/Pattern/RenderContext（应用进程/UI 线程）
→ ③ RSNode + 绘制指令序列化，IPC 发送（应用进程 → RS 进程）
→ ④ Render Service：命令回放、GPU 绘制、图层合成（render_service 进程）
→ ⑤ 送显：HWC/显示驱动（内核层）
```

逐环节展开。

**① 声明式代码到组件树。** 我们写的每个自定义组件（`@Component struct Foo`）在方舟运行时里对应一个 JSView 对象。`build()` 函数执行时调用系统组件（`Text`、`Column` 等）的创建接口，这些调用经前端桥接层（`frameworks/bridge/declarative_frontend`）落到 C++ 侧，创建出组件树。状态管理的依赖收集也发生在这里：build 过程中读了哪些状态变量，框架记下"这个组件依赖这些变量"（2.3 节展开）。

**② FrameNode 与 Pattern。** C++ 侧的核心结构是 FrameNode（`frameworks/core/components_ng/base/frame_node.cpp`）。每个 FrameNode 关联一个 Pattern（如 `ListPattern`、`TextPattern`），Pattern 封装了该类组件的布局算法、事件处理和绘制逻辑。FrameNode 还持有一个 RenderContext（`frameworks/core/components_ng/render/adapter/rosen_render_context.cpp`），负责把"背景色、圆角、变换、模糊"这类渲染属性翻译成底层 RSNode 的属性。

**③ 绘制指令跨进程。** ArkUI 应用进程不直接画屏。每个 FrameNode 的 RenderContext 背后对应一个 RSNode（Render Service Node），树结构与 FrameNode 树镜像。每帧末尾，UI 线程把本帧改动的 RSNode 属性序列化成事务数据发给 render_service 进程——Trace 里看到的 `H:FlushMessages → H:SendCommands → H:MarshRSTransactionData` 就是这一步。官方启动分析文档对这三个点的描述是：FlushMessages 代表绘制消息，SendCommands 代表发送绘制指令给 Render Service 进程，MarshRSTransactionData 代表绘制指令已序列化发出。

**④ Render Service 绘制合成。** render_service 进程收到事务后更新自己维护的 RSNode 树，在 Vsync 驱动下遍历树生成 Skia 绘制命令、提交 GPU，再把各窗口图层合成。这一半是 2.4 节的主题。

**⑤ 送显。** 合成结果交给硬件合成器（HWC）或经 GPU 最终合成后由显示驱动送上屏幕。

[图：渲染全链路架构图。左侧应用进程框内自上而下：ArkTS @Component（build）→ JSView/前端桥接 → FrameNode 树（标注 Pattern、RenderContext）→ RSNode 代理 + 事务序列化（标注 Trace 点 H:MarshRSTransactionData）；中间 binder IPC 箭头；右侧 render_service 进程框内：RSNode 树 → RSMainThread（命令处理）→ RSUniRenderThread（Skia/GPU 绘制）→ 图层合成 → HWC → 屏幕。底部标注时间轴：整条链路必须在一个 Vsync 周期内完成才不丢帧。]

## 一帧的驱动：PipelineContext 与 FlushVsync

把环节 ①②③ 串起来的总控是 PipelineContext（`frameworks/core/pipeline_ng/pipeline_context.cpp`）。窗口系统把 Vsync 信号回调给它（`OnVsyncEvent`），它在 UI 线程执行 `FlushVsync()`，一帧的应用侧工作就此展开。`FlushVsync` 里的关键调用序列：

```cpp
// frameworks/core/pipeline_ng/pipeline_context.cpp，FlushVsync() 精简示意
SetVsyncTime(nanoTimestamp);        // 记录本帧 Vsync 时间戳
ProcessDelayTasks();                // 执行延迟任务
FlushAnimation(nanoTimestamp);      // 推进动画
FlushTouchEvents();                 // 分发排队的触摸事件
FlushBuild();                       // 重建脏组件（Build 阶段）
taskScheduler_->FlushTask();        // Measure/Layout/Render 任务
FlushMessages();                    // 把 RS 事务发给 render_service
```

这段代码决定了 Trace 的形态：Frame 模板里应用侧一帧的 Trace 结构就是这几个函数的执行区间，`UITaskScheduler::FlushTask` 对应 `FlushTask()`，`FlushMessages` 对应最后一步。记住这个对应关系，拿到任何一帧的 Trace 都知道自己站在流水线哪一段。

## 关键设计：UI 线程与渲染进程分离

鸿蒙渲染架构与 Android（HWUI + SurfaceFlinger）最大的表面差异，是渲染进程只有**一个**：所有应用的绘制指令都发给同一个 render_service 进程统一绘制、统一合成，而不是每个应用自己 GPU 绘制后由 SurfaceFlinger 合成。这就是官方说的"统一渲染"模式（2.2 节详述）。

这个设计带来三个直接后果：

- **动画可以在 RS 侧独立运行。** 属性动画、转场动画只把起点终点和曲线发给 RS，之后每帧由 RS 自己插值，应用 UI 线程被占住时动画依然流畅——这也是官方动画文档说属性动画"性能较好、UI 侧不感知实时渲染值"的原因；与之相对的帧动画（ohos.animator）每帧都要应用侧算值，标注"性能：较差"。
- **应用进程与 RS 进程互相影响的方式变了。** 应用侧慢了，RS 拿到的是旧帧（AppDeadlineMissed）；RS 自己慢了，所有应用一起卡（RenderDeadlineMissed）。丢帧分析的第一步永远是分清这两种红帧。
- **跨进程开销是固有的。** 每帧一次事务序列化 + IPC，节点树越大、改动越多，这段开销越大。大列表每帧重建大量组件时，`MarshRSTransactionData` 的耗时会明显上涨。

## 实际工作中怎么用

**看 Trace 先定位环节。** 应用侧红帧：展开该帧，按 `FlushVsync` 的子区间对号入座——动画区间长查动画，Build 区间长查组件重建（2.3 节），FlushTask 长查布局，MarshRSTransactionData 长查节点树规模。RS 侧红帧去 render_service 泳道（2.4 节）。

**确认链路事实，用源码校准。** 上面每个环节都有开源实现可查：`openharmony/arkui_ace_engine` 的 `frameworks/core/pipeline_ng/`（管线）、`frameworks/core/components_ng/`（FrameNode/Pattern）、`frameworks/core/components_ng/render/adapter/`（对接 Rosen）；`openharmony/graphic_graphic_2d`（RSNode、render_service 进程）。读机制章节时对着代码读，比背文档可靠。

## 版本演进

- API 9–11：ArkUI 新一代组件框架（components_ng，即"NG 框架"）已是默认，声明式范式全面替代早期类 Web 范式；render_service 统一渲染架构成形。
- API 12（5.0）：状态管理 V2 引入（2.3 节），前端桥接增加 ArkTS 静态前端；FrameNode/NodeController 等命令式节点 API 开放，允许绕过声明式直接操作节点树。
- API 20+（6.0）：组件库逐步拆分为独立 so 按需加载（官方仓库说明中 libarkui_*.z.so 的拆分），首帧加载路径缩短；同层渲染（Web/XComponent 与 ArkUI 控件混合）持续优化。

## 常见问题与误区

**"ArkUI 是声明式框架，所以没有布局过程"。** 错。声明式只改变我们描述 UI 的方式，底层 Measure/Layout 一样不少，Layout 嵌套过深照样是丢帧主因。

**"动画卡顿就是 GPU 不行"。** 先看动画跑在哪一侧。属性动画在 RS 侧插值，卡说明 RS 忙；animator 帧动画在应用侧算值，卡说明主线程忙。两者的优化方向完全不同。

## 参考资料

- OpenHarmony 仓库 `openharmony/arkui_ace_engine`：`frameworks/core/pipeline_ng/pipeline_context.cpp`（FlushVsync）、`frameworks/core/components_ng/base/frame_node.cpp`、`frameworks/core/components_ng/render/adapter/rosen_render_context.cpp`
- OpenHarmony 仓库 `openharmony/graphic_graphic_2d`（RSNode 与 render_service）
- 华为开发者文档：《启动分析案例》（ide-profiler-launch-case：FlushMessages/SendCommands/MarshRSTransactionData 的含义）
- 华为开发者文档：《丢帧分析》（bpta-zhenlv：应用侧与 Render Service 侧职责划分）
- 华为开发者文档：《Animator 与动画方式对比》（arkts-animator：帧动画"性能：较差"、属性动画"性能：较好"）
- 华为开发者文档：《自定义节点 FrameNode》（arkts-user-defined-arktsnode-framenode）
