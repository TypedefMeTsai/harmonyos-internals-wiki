# 2.2 Vsync、统一渲染与一帧的生命周期

## 从一条红帧说起

在 Frame 模板里，丢帧被画成红色的帧条。官方丢帧分析文档把故障模型分成两种：**AppDeadlineMissed**（应用侧没按时交卷）和 **RenderDeadlineMissed**（Render Service 侧没按时画完）。这两种红帧的判定都建立在同一个时间基准上——Vsync。不理解 Vsync 是怎么分发、一帧的预算怎么算，就谈不上分析任何流畅性问题。

## Vsync：帧节拍的来源

Vsync（垂直同步信号）由显示硬件按刷新率产生，是整机渲染管线的节拍器。每个周期对应一帧的预算时间：

| 刷新率 | Vsync 周期 |
| --- | --- |
| 60 Hz | 16.7 ms |
| 90 Hz | 11.1 ms |
| 120 Hz | 8.3 ms |

这个预算就是"App 侧帧耗时需要在 8.3ms 以内"这类官方说法的来源（120Hz 时）。鸿蒙的 Vsync 分发由图形栈的 VsyncGenerator/VsyncDistributor 负责：它不是把硬件信号广播给所有进程，而是**按需分发**——应用有界面更新需求（注册了帧回调）才收到应用侧 Vsync，render_service 则按显示节奏收到自己的 RS 侧 Vsync。应用还可以通过 DisplaySync 请求不同的期望帧率（比如动画声明 30fps），系统据此调节发给它的 Vsync 频率，这是帧率级省电的手段。

Frame 模板里的相关泳道：`Display Vsync`（显示侧信号）、`DisplaySync_cb`（应用侧收到的 Vsync 回调）。官方 IDE 文档提到，在 DisplaySync_cb 子泳道可以查看应用侧组件的帧率和渲染时间。

## 统一渲染：一个进程画所有窗口

2.1 节说过，鸿蒙所有应用的绘制指令都汇到 render_service 进程。这套"统一渲染"模式下，一帧的完整循环是：

1. 应用侧收到 Vsync（`H:ReceiveVsync`），执行下文七阶段，产出绘制事务发给 RS；
2. RS 侧收到自己的 Vsync（render_service 泳道里的 `H:ReceiveVsync`），处理各应用发来的事务（`RSMainThread::ProcessCommandUni`），遍历 RSNode 树绘制（RenderFrame）、合成、送显（PresentFence）。

官方丢帧文档对 RS 侧的描述是：Render Service 的 RenderThread 线程在 Vsync 信号触发下进行 UI 绘制，绘制过程包含动效、描画和提交三个阶段。两侧是两个相位错开的流水线：应用侧第 N 帧的产物，由 RS 在下一个 Vsync 周期消费。所以一帧端到端占用约两个 Vsync 周期，这也是为什么单帧预算必须卡在一个周期以内——任何一侧超时，都挤占下一个周期的渲染时间，产生连锁丢帧（官方 IDE 帧率文档里 vsync2/vsync4 挤占后续周期的图示说的就是这个现象）。

## 一帧的七个阶段

应用侧从收到 Vsync 到交出绘制事务，官方帧率分析资料把这段工作划分为七个阶段：

```
Animation → Events → UpdateUI → Measure → Layout → Render → SendMessage
```

对应到 2.1 节的 `FlushVsync()` 代码：

- **Animation**：`FlushAnimation(nanoTimestamp)`，推进所有进行中的动画，计算本帧动画值。animator 帧动画的回调就在此阶段执行。
- **Events**：`FlushTouchEvents()` 等，分发本周期到达的输入事件（触摸、拖拽）。事件回调里改状态变量，产生的脏节点进入下一阶段。
- **UpdateUI**：`FlushBuild()`，重建脏组件——执行脏自定义组件的 build、更新组件属性、给属性变化的组件标脏。Trace 里常见的 `JSView: ExecuteRerender`、`H:Builder:BuildLazyItem` 落在这一阶段。
- **Measure**：测量。`taskScheduler_->FlushTask()` 的前半段，自底向上确定每个 FrameNode 的尺寸。
- **Layout**：布局。确定每个节点的位置。Trace 点 `FlushLayoutTask`。
- **Render**：生成绘制指令——遍历脏节点的 RenderContext，把属性变化写入对应 RSNode。Trace 点 `FlushRenderTask`。
- **SendMessage**：`FlushMessages()`，把本帧 RS 事务序列化并发给 render_service，即 `H:SendCommands` / `H:MarshRSTransactionData`。

七阶段的含义很直白：**任何一帧的耗时是这七段之和，优化就是找到最长的那段对症下药。** 滑动丢帧场景里，最常见的长段是 UpdateUI（组件重建太多）和 Measure/Layout（布局嵌套太深）；SendMessage 显著变长通常意味着节点树过大或每帧改动面过大。

## 在 Trace 中看一帧：正常与异常

**正常帧**（120Hz）：应用侧 `H:ReceiveVsync` 区间总长 < 8.3ms，七段结构清晰，末尾 `MarshRSTransactionData` 短促；render_service 泳道同相位的 `H:ReceiveVsync` 也在预算内，PresentFence 准时出现。

**AppDeadlineMissed**：应用侧区间跨过一个 Vsync 边界。官方 FAQ 给过一个实测案例：120Hz 下 `H:ReceiveVsync` 整体耗时 13.4ms，超过 8.3ms 预算，判定丢帧。此时往七阶段里钻，找最长的子区间。

**RenderDeadlineMissed**：应用侧准点，render_service 泳道超时。另一个官方 FAQ 案例演示了反向排除法：render_service 线程 `H:ReceiveVsync` 处理最多 7.6ms 未超 8.3ms，因此排除 RS 侧，回头查应用进程。这种"先证明哪侧没病"的思路，比直接猜要省时间。

还有一个容易误读的点：应用侧的帧耗时统计在 Details 面板里能看到 Expected Duration（期望耗时，120fps 对应 8ms330μs）与 Actual Duration 的对比，实际耗时远超期望即被识别为卡顿帧——官方 IDE 案例里的示例帧实际耗时 44ms54μs。

## 实际优化方向

- **预算意识**：先确认设备当前刷新率（Display Vsync 泳道会显示），90Hz 设备的预算是 11.1ms 而不是 8.3ms，别拿错尺子。
- **分侧处理**：AppDeadlineMissed 优化应用主线程（减少重建、异步化耗时逻辑）；RenderDeadlineMissed 减少界面复杂度、图层数量和 GPU 负载（2.4 节）。
- **帧率匹配**：对不要求满帧的场景（如进度动画），用 `expectedFrameRateRange` 声明期望帧率，系统降帧分发 Vsync，直接省电。
- **连续丢帧看连锁**：单帧超时挤占后续周期的现象，在 Trace 上表现为红帧成片出现，修根因时找第一帧红帧的长段，而不是最后一帧。

## 版本演进

- API 9–11：统一渲染架构与 Frame 模板工具链成形，AppDeadlineMissed/RenderDeadlineMissed 故障模型确立。
- API 12（5.0）：DisplaySync 期望帧率接口完善，动画可声明 `expectedFrameRateRange`；Vsync 按需分发策略在更多场景生效。
- API 20+（6.0）：帧率分析工具进一步细分泳道（DisplaySync_cb 等），Web 自渲染的丢帧判定（RosenWeb buffer 为 0 判丢帧）写入官方最佳实践。

## 参考资料

- 华为开发者文档：《丢帧分析/帧率优化概述》（bpta-zhenlv、bpta-optimization-overview：AppDeadlineMissed 与 RenderDeadlineMissed 故障模型、RS 绘制三阶段）
- 华为开发者文档：性能 FAQ（faqs-performance-53/6：120Hz 下 H:ReceiveVsync 13.4ms 超时案例、render_service 7.6ms 排除案例）
- 华为开发者文档：《Frame 会话使用》（ide-insight-session-frame、ide-frame-case：Expected/Actual Duration、Display Vsync 与 DisplaySync_cb 泳道）
- 华为开发者文档：《DisplaySync 与期望帧率动画》（displaysync-animation）
- 华为开发者文档：《Web 帧率性能分析》（bpta-web-frame-rate-performance-analysis：VSyncGenerator 与 RosenWeb buffer 判定）
- OpenHarmony 仓库 `openharmony/arkui_ace_engine`：`frameworks/core/pipeline_ng/pipeline_context.cpp`（FlushVsync）；`openharmony/graphic_graphic_2d`（Vsync 分发与 RS 渲染）
