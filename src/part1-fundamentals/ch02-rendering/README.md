# 第 2 章：ArkUI 渲染系统

鸿蒙应用性能问题里，渲染类问题占了绝对大头：丢帧、滑动卡顿、转场掉帧、动效不跟手。要分析它们，绕不开一个问题——我们写下的 `Text("hello").fontSize(16)` 是怎样变成屏幕上的像素的？

这条链路比多数人想象的长：ArkTS 声明式代码先在应用进程内经 ArkUI 框架构建出 FrameNode 树；状态变化触发脏节点重建与布局；每帧由 Vsync 信号驱动，把绘制指令序列化后跨进程发给 Render Service；Render Service 在独立进程里用 GPU 绘制、合成，最后交给硬件合成器送显。链路上任何一环超时，用户看到的就是一帧卡顿——而 Frame 模板里的 AppDeadlineMissed 和 RenderDeadlineMissed 两种红帧，分别对应链路的前半段和后半段。

## 各小节地图

- **2.1 渲染架构总览**：完整链路的地图。从 @Component 声明到 FrameNode、Pattern、RenderContext、RSNode，再到 Render Service 进程。先把这一节读通，后面四节都是它的局部放大。
- **2.2 Vsync、统一渲染与一帧的生命周期**：帧节拍的来源。Vsync 信号如何分发，90Hz 下 11.1ms 的预算意味着什么，一帧在应用侧的七个阶段（Animation/Events/UpdateUI/Measure/Layout/Render/SendMessage）各自对应哪些 Trace 点。
- **2.3 状态管理、脏区刷新与组件重建**：改一个 @State 变量之后系统做了什么。依赖收集、脏节点标记、批量合并、属性级局部刷新，以及组件复用（@Reusable）为什么能把丢帧率从 3.7% 降到 0%。
- **2.4 Render Service、RSNode 与 HWC 送显**：链路的另一半，在系统侧进程里。RSNode 树、脏区管理、GPU 合成与 HWC 直通两条送显路径，以及为什么视频上盖一个模糊控件会让功耗涨四分之一。
- **2.5 自渲染管线**：游戏、相机、跨平台引擎（RN/Flutter）自带渲染器时怎么接入：XComponent、NativeWindow、NativeVSync，以及自渲染内容与系统帧率同步的正确姿势。

## 阅读顺序

建议按 2.1 → 2.2 → 2.3 → 2.4 → 2.5 顺序读。2.1 建坐标系，2.2 建立"帧"的时间概念，2.3 讲应用侧最常见的性能变量（状态刷新），2.4 讲系统侧，2.5 是相对独立的专题。读完本章再去看第二部分的丢帧案例（第 7 章），每一个分析步骤都能找到出处。
