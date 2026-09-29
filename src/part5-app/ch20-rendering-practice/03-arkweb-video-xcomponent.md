# 20.3 ArkWeb、视频与 XComponent 场景

## ArkWeb 的丢帧模式：它有自己的渲染进程

ArkWeb 页面不是 ArkUI 组件树的一部分。Web 内容在独立的 Webview 渲染进程中布局、绘制、栅格化，产出的 buffer 再交给 Render Service 与应用的其它图层合成。所以 ArkWeb 丢帧要同时看两条线：Web 渲染进程内部发生了什么，以及 buffer 有没有按时到达 RS。

DevEco Profiler 的 ArkWeb 泳道把 Web 相关线程的 Trace 聚合在一起，几个关键 Trace 点（《Profiler ArkWeb 分析》文档）：

- **H:RosenWeb**：记录准备提交给 Render Service 做统一渲染的数据量。这是判断"Web 侧有没有产出"的直接证据——**某个 Vsync 周期内 RosenWeb = 0，意味着图形侧拿不到 buffer，这一帧只能沿用旧内容或直接丢帧**。
- **JSView:ExecuteRerender 一类的 JS 执行 Trace**：Web 前端框架的重渲染执行。长时间占用意味着 JS 侧阻塞，绘制指令产不出来。

官方《Web 页面帧率性能分析》最佳实践给了一个标准故障链：用户在滑动 Web 页面的过程中，应用自身业务逻辑长时间阻塞主线程 → Vsync 到来时 Web 手势无法正常下发 → 这个 Vsync 周期内 ArkWeb 无法完成绘制、没有 buffer 产出 → 下一个 Vsync 时 RosenWeb = 0 → 丢帧。读 ArkWeb 泳道时按这个链条倒着查：先确认丢帧周期 RosenWeb 是否为 0，是 0 就回查同周期 JS 线程在忙什么。

渲染模式也影响合成行为：默认的**异步渲染模式**（RenderMode.ASYNC_RENDER）下 Web 组件作为图形 surface 节点独立送显，官方建议在仅由 Web 组件构成的页面中使用以提高性能降低功耗（注意此模式下 Web 组件高度不能超过 7680 物理像素，超过会白屏）；**同步渲染模式**（SYNC_RENDER）下 Web 作为 canvas 节点跟随系统组件一起送显，可以渲染更长内容，但性能消耗增加且不支持 DSS 合成。两种模式不支持动态切换。

### Web 启动与跳转加速：预创建、预渲染、预连接

Web 渲染进程的冷启动开销远大于普通页面。官方《Web 开发优化》最佳实践给出的数字：**预创建一个空白 ArkWeb 组件大约消耗 200MB 内存**——这个数字同时说明了为什么预创建有效（省掉的是一整个进程的初始化）和为什么不能滥用（每多一个存活 Web 组件就是 200MB 量级的预算）。正确姿势：

1. 应用全局共享一个 Web 渲染进程，确保至少有一个 Web 组件处于活动状态——所有 Web 组件销毁时渲染进程会终止，下次重建又是完整开销。
2. 用离线 Web 组件做**预渲染**：页面创建时建好离线组件并加载目标网页执行预渲染，跳转时直接复用预渲染完成的组件，比现场加载快得多（《Web 离线模式》文档的对比演示）。
3. 再往前一层是网络：Web 页面依赖的接口用预连接/预解析提前完成 DNS 与 TLS 握手。
4. 有返回-前进场景的页面可开启 BFCache（在 initializeWebEngine 之前调用 enableBackForwardCache），返回时不重新加载。

验证：Profiler 的 ArkWeb 泳道对比预渲染前后的页面可交互时间；Memory 会话确认预创建组件数量没有失控。

## 视频与自渲染图层：和统一渲染管线的接缝

视频播放（AVPlayer 绑 XComponent 的 surface）、游戏、相机预览这类场景走的是自渲染管线：内容生产者直接往 surface 里填 buffer，不经 ArkUI 组件树绘制，最终由 RS/HWC 与其它图层合成。性能问题大多出在"接缝"上。

### 视频图层与模糊控件交叠：HWC 失效的典型案例

20.2 讲过机制：自渲染图层上方的 UI 控件使用模糊等高阶视效时，模糊要实时读背景图层内容，HWC 无法使能，RS 改用 GPU 合成——功耗上升，发热和卡顿概率随之上升。视频页面的典型踩坑组合：视频区域上叠一个模糊底衬的标题栏或弹幕控制条。

官方《HWC 自渲染图层分析》最佳实践的建议直接：**移除模糊等高阶视效，或调整控件位置避免与自渲染内容交叠**。具体做法：把控制条的模糊背景换成半透明纯色；让模糊控件完全不覆盖视频区域（布局上避开）；必须保留模糊时评估它覆盖的像素范围，范围越小 GPU 合成代价越低。

验证：对比改动前后 GPU 合成是否退回 HWC 使能（Profiler 图层信息/hidumper 图形模块），同时看 GPU 负载与设备温升。

### XComponent 与同层渲染

XComponent 是 ArkUI 给自渲染内容预留的"窗口"，两种类型：SURFACE 类型的内容独立成层（适合视频、游戏，合成路径短），TEXTURE 类型的内容作为纹理参与 ArkUI 统一绘制（适合需要和声明式组件混合变换的场景）。选型原则：全屏视频、相机预览用 SURFACE；需要被 ArkUI 动画、裁剪、圆角包裹的自绘内容用 TEXTURE。

Web 里的视频有专门的**同层渲染**机制：通过 `enableNativeEmbedMode(true)` 开启后，可以把原生组件（如基于 XComponent 的播放器）嵌入到网页的 video 标签区域渲染，播放走原生 AVPlayer 而不是 Web 内核的软件解码路径。官方还提供了更完整的"应用接管 Web 媒体播放"方案（enableNativeMediaPlayer + onCreateNativeMediaPlayer 回调）：ArkWeb 内核发现 video 标签时把播控回调给应用侧的原生播放器实现，原生播放器的状态通过 NativeMediaPlayerHandler 回传给内核。收益是硬解、省内存、播控体验与原生一致；代价是要自己实现播放器的完整桥接（play/pause/seek/volume 等回调都要接）。

视频性能还有两个来自体验规范的硬指标可以对照自查：在线长视频起播时延应 ≤800ms；在线短视频滑动切换新视频起播时延应 ≤230ms（《应用性能体验建议》时延章节）。短视频场景的 230ms 基本决定了必须做播放器实例池和下一条预加载，现场创建播放器达不到这个数。

### 验证清单

- Web 丢帧：ArkWeb 泳道看 RosenWeb 是否为 0、JS 线程长任务；Frame 泳道确认丢帧归属（App 侧还是 Web 侧）。
- Web 启动：ArkWeb 泳道看页面可交互时间；Memory 会话看 Web 渲染进程内存。
- 视频合成：Profiler 图层信息看 HWC/GPU 合成方式；GPU 负载与功耗回归。
- 起播时延：HiTraceMeter 在"点击播放 → 首帧解码上屏"打点，对照 ≤800ms / ≤230ms 的规范值。

## 参考资料

- 官方文档：《Profiler ArkWeb 分析》（ide-profiler-arkweb）
- 官方文档：《Web 页面帧率性能分析》（bpta-web-frame-rate-performance-analysis）
- 官方文档：《Web 开发优化》（bpta-web-develop-optimization）
- 官方文档：《Web 离线模式（预渲染）》（web-offline-mode）
- 官方文档：《Web 渲染模式》（web-render-mode）
- 官方文档：《Web 同层渲染》（web-same-layer）、《应用接管 Web 媒体播放》（app-takeovers-web-media）
- 官方文档：《高效使能 HWC》（bpta-utilize-hwc-efficiently）、《HWC 自渲染图层分析》（bpta-hwc-self-rendering-layer-analysis）
- 官方文档：《应用性能体验建议》时延章节（视频起播指标）
