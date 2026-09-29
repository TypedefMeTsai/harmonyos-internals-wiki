# 2.5 自渲染管线：XComponent、NativeWindow 与 NativeVSync

## 为什么需要自渲染

ArkUI 的声明式管线覆盖不了所有画面：游戏引擎要自己控制每一帧的绘制，相机预览要把解码器的输出直接上屏，React Native/Flutter 这类跨平台框架带着自己的渲染器，地图 SDK 也是。这些场景的共同点是——**画面内容不由 ArkUI 组件树产生**，但它们又必须嵌在 ArkUI 的页面里，和按钮、文本共存。

鸿蒙的答案是自渲染管线：ArkUI 在组件树上开一个"洞"（XComponent），把一块独立的生产者 buffer 交给应用自己的渲染代码，画完交给 Render Service 统一合成。这条路走对了，性能和原生组件无异；走错了——官方 XComponent 渲染问题指导里最常见的症状是帧率与系统不同步导致的卡顿，表现像丢帧，根因却完全不在 ArkUI。

## XComponent：组件树上的自渲染入口

XComponent 是 ArkUI 提供的自渲染组件，两种类型：

- **SURFACE**：独立的 buffer 生产者通道，内容可以走 HWC 直通（见 2.4 节），视频、相机、游戏的首选；
- **TEXTURE**：内容作为纹理参与 ArkUI 场景合成，适合做滤镜、离屏渲染结果展示。

最基本的用法是拿 SurfaceId，再交给 Native 侧：

```typescript
XComponent({ type: XComponentType.SURFACE, controller: this.xcController })
  .onLoad(() => {
    // 拿 surfaceId，传给 Native 层绑定渲染上下文
    this.surfaceId = this.xcController.getXComponentSurfaceId();
    nativeRender.init(this.surfaceId);
  })
  .width('100%').height(240)
```

API 19 起还可以更直接：用构造参数创建 XComponent 后，通过 `getUIContext().getAttachedFrameNodeById()` 拿到它的 FrameNode，用 NDK 接口 `bindNode` 做 Surface 生命周期绑定——适合用 ArkUI_NativeModule 体系开发的应用。相机预览就是标准场景：官方相机指南里 PreviewOutput 的 surfaceId 正是从 XComponentController 取的。

## NativeWindow 与 Buffer 流转

SURFACE 模式下，Native 侧拿到的是一个 NativeWindow（`OH_NativeWindow*`），它就是 buffer 队列的生产者句柄。官方 NDK 文档把流转概括为一条环形队列：

```
应用 RequestBuffer 获取空闲帧 → 应用生产帧数据（GL 绘制/解码输出）
→ FlushBuffer 提交到 BufferQueue → 系统渲染侧 AcquireBuffer 取帧
→ 合成上屏 → 系统 ReleaseBuffer 归还
```

Native 侧的典型循环：

```cpp
// 向 NativeWindow 申请一块 buffer
OH_NativeWindow_NativeWindowRequestBuffer(nativeWindow, &buffer, &fenceFd);
// ……用 EGL/OpenGL ES 把内容画进 buffer……
// 提交，交给系统合成
OH_NativeWindow_NativeWindowFlushBuffer(nativeWindow, buffer, fenceFd, region);
```

两个细节决定体验。**其一，buffer 数量**：队列深度不够时生产者会被 RequestBuffer 阻塞，渲染线程停顿。**其二，提交节奏**：FlushBuffer 的时机要踩上系统合成的节拍，否则这帧要等到下一轮才被消费——这就引出 NativeVSync。

## NativeVSync：和系统帧率同步

自渲染内容最终由 Render Service 统一合成，RS 有自己的 Vsync 节拍（2.2 节）。如果应用自渲染线程"画完就提交"，不管节拍，就会出现两种问题：产快了，buffer 堆积排队，延迟增加；产慢了，RS 合成时取不到新帧，画面不动。

官方《XComponent 渲染问题指导》对此说得很明确：为了确保不同来源的绘制效果在同一时间节点完成绘制、合成与显示，Render Service 依赖 VSync 实现全局统一的信号收发；**如果在调用 `OH_NativeWindow_NativeWindowFlushBuffer()` 提交 Buffer 时没有申请 NativeVsync，会导致应用的绘制帧率与系统帧率不同步，出现卡顿**。解决方案之一就是在提交 Buffer 前申请 VSync。

正确姿势是用 NativeVSync 驱动生产循环：

```cpp
// 创建 vsync 实例并注册回调，每次回调里画一帧、提交一帧
OH_NativeVSync* nativeVsync = OH_NativeVSync_Create("renderThread");
OH_NativeVSync_RequestFrame(nativeVsync, OnVsync, this);
// 回调末尾再次 RequestFrame，形成逐帧循环
```

帧率诉求不同步的场景（游戏锁 60、屏幕 120），用 `OH_NativeXComponent_SetExpectedFrameRateRange()` 声明期望帧率范围，让系统按这个范围给节拍，既不掉帧也不白画。官方低功耗文档也明确推荐：通过 XComponent 在 Native 侧申请独立的绘制帧率用于游戏等自绘制内容。

## 在 Trace 中看自渲染

自渲染线程不是 ArkUI 的 UI 线程，Trace 形态不同，容易误读。官方 Vsync 功耗优化文档给过一个辨识方法：应用进程下 `OS_VsyncThread` 线程发出的 `H:ReceiveVsync`（name 是自定义的，比如 TaroAnimation、dirty）是自渲染帧回调；同一帧里如果 `FlushRenderTask` 没有打印需要刷新的 ArkUI 组件，但 `FlushMessage` 下仍有 `H:MarshRSTransactionData` 送出，说明绘制内容来自自渲染通道而非 ArkUI 组件树。分析这类页面的"丢帧"时，先分清卡在自渲染生产端（OnVsync 回调太长）还是合成端。

跨平台框架（RN/Flutter/Taro）的页面是混合形态：框架自己的渲染线程走自渲染通道，页面里嵌的 ArkUI 原生组件走声明式管线，两条通道共用系统 Vsync。官方文档指出 NativeVsync 被广泛用于 React Native 及 C-API 开发的应用，这些框架靠持续请求 Vsync 拉起业务内容——所以这类应用静置页面下 OS_VsyncThread 仍有规律回调，属预期行为，但回调里做了无效绘制就是功耗问题。

## 常见问题与排查

**症状一：画面一顿一顿，但 GPU 占用不高。** 多半是生产节奏没踩拍：没接 NativeVSync，或期望帧率与系统帧率错配。按上文接好 Vsync 驱动循环。

**症状二：操作延迟感强（画面慢半拍）。** buffer 队列堆积。检查是否一帧里提交了多份 buffer、解码后处理是否太重（官方建议提高生产端速度，如硬件解码、优化后处理算法）。

**症状三：功耗高。** 查两点——自渲染图层上是否叠了高阶视效破坏 HWC 直通（2.4 节）；静置时生产端是否还在满帧空画，该降帧时用 SetExpectedFrameRateRange 或停掉 RequestFrame 循环。

**症状四：同层混排异常。** XComponent 与 ArkUI 控件交叠、层级错乱时，先确认是否需要同层渲染能力（官方同层渲染文档列了支持同层的自绘制组件：XComponent、Canvas、Video、Web），并检查不支持同层的属性（特效绘制合并等）是否被误用。

## 版本演进

- API 9–11：XComponent SURFACE/TEXTURE 与 NativeWindow NDK 接口成形；相机、视频场景标准接入。
- API 12（5.0）：同层渲染推广，自渲染内容与 ArkUI 控件混合摆放的限制减少；SetExpectedFrameRateRange 等帧率控制接口补齐。
- API 19+：FrameNode 绑定式 XComponent（bindNode）开放，NDK 体系应用接入更直接；官方 XComponent 渲染问题指导、Vsync 功耗优化等最佳实践文档陆续发布。

## 参考资料

- 华为开发者文档：《XComponent 渲染问题指导》（bpta-xcomponent-render-problem-guide：未申请 NativeVsync 导致帧率不同步卡顿）
- 华为开发者文档：《XComponent NDK 开发指导》（napi-xcomponent-guidelines：RequestBuffer→FlushBuffer→AcquireBuffer→ReleaseBuffer 流转、bindNode 用法）
- 华为开发者文档：《Vsync 功耗优化》（bpta-vsync-power-optimization：OS_VsyncThread、NativeVsync 被 RN/C-API 框架使用）
- 华为开发者文档：《自渲染与自定义绘制能力对比》（arkts-user-defined：XComponent 适用游戏引擎、地图、相机场景）
- 华为开发者文档：《低功耗体验设计》（bpta-power-consumption-experience：XComponent 申请独立绘制帧率）
- 华为开发者文档：`native_interface_xcomponent.h`（OH_NativeXComponent_SetExpectedFrameRateRange）、相机预览指南（camera-preview）
- 华为开发者文档：《同层渲染》（web-same-layer：支持同层的自绘制组件清单）
- OpenHarmony 仓库 `openharmony/graphic_graphic_2d`（BufferQueue/NativeWindow 实现）
