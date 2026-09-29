# 14.1 Frame 会话：丢帧与帧耗时分析

## 这个工具解决什么问题

滑动不顺畅、页面交互延迟、动效不流畅——这些卡顿类问题，DevEco Profiler 的 Frame 会话提供一条龙的分析路径：录制卡顿过程中的关键数据，识别丢帧位置、判定丢帧侧（App 还是 RS）、逐层深入到函数级耗时。本书第 7、8 章的方法论全部以它为载体，本节把工具操作本身讲完整。

基本流程：DevEco Studio 底部 Profiler 面板 → 选择设备与应用进程 → 选择 Frame 模板创建 Session → 开始录制并在设备上复现场景 → 停止录制等待解析 → 时间线上框选分析。详细操作参考官方《性能问题定位：深度录制》；历史数据可通过会话区 Open File 导入。

## 泳道全景

Frame 模板支持的泳道（出处：《Frame 分析》）：Anomaly、User Events、Frame、ArkUI Component、ArkUI State、User Trace、ArkTS Callstack、Callstack、Network Traffic、Network Request、Energy、CPU Core、Process。其中 ArkUI Component 和 ArkUI State **默认不采集，需在录制前的泳道配置中手动勾选**——这是新手最常踩的坑，录完发现没有组件数据，只能重录。

## Frame 泳道：红绿帧与丢帧判定

Frame 主泳道顶部显示 GPU 使用率，展开后是 RS Frame 和 App Frame 两条子泳道，分别承载 Render Service 侧和 App 侧的帧数据。判读规则（出处：《Frame 分析》《帧率问题分析》）：

- 正常完成渲染的帧为绿色，卡顿帧为红色；异常帧为黄色。
- 红色帧中，期望结束时间点之前的部分为浅红色（两条白色竖线区间），超出的部分为深红色。
- 判定标准：帧的实际结束时间晚于期望结束时间。期望耗时由帧率决定，60fps 对应 16.6ms。
- Jank Type 三值：No Jank、AppDeadlineMissed（App 侧卡顿）、RenderDeadlineMissed（RS 侧卡顿）。
- App 帧与 RS 帧可能一对多关联：多个 App 帧可提交到同一个 RS 帧。

[图：Frame 泳道局部。App Frame 子泳道上绿色帧为主，中间一个红色帧带两条白色竖线（Expected Start / Expected End），深红段拖出期望结束线；对应 RS Frame 上有关联箭头指向。]

**定量统计**：框选时间段后，选中 Frame 主泳道，Statistics 区按进程给出卡顿率、卡顿次数、最大连续卡顿次数、最大/平均卡顿耗时。点进程后的跳转按钮进 Frame List，列出每帧的起始时间、总耗时、GPU 耗时、Jank Type，可按类型排序，点任意帧跳转定位。选中单帧后，Frame 详情区给出 VSync 编号、App 侧持续时间、业务逻辑耗时、RS 侧持续时间、GPU 持续时间、总耗时、Jank Type 及可能的卡顿原因；App Frame 泳道还有 Non UI 区域，显示耗时最大的非 UI 函数。框选 App Frame 或 RS Frame 子泳道时，泳道上还会出现 FPS 标记，显示框选范围内的帧率统计。

**连续卡顿量化**：Lost Frames 子泳道展示时间段内丢帧数（六舍七入取整），Hitch Time 子泳道展示卡顿时长——计算方式为相邻两帧间隔减去单帧耗时，结果大于单帧耗时 70% 即记为卡顿。这两个指标比数红帧更接近用户体感。

**帧页面布局回放**（DevEco Studio 5.1.0 起）：选中任一帧，Details 区点 Open Layout 可直接在 ArkUI Inspector 中打开该帧的页面布局（arkli 文件），或 Download Layout 下载后手动导入。前提：操作前把应用进程置于前台，才能正确回放全量渲染数据。6.1.0 Beta1 起该功能有 Frame Layout 开关，默认关闭。

## 分析三件套：Callstack、Component、Anomaly

定位到红帧只是开始，解释红帧靠三条辅助泳道。

**ArkTS Callstack**：ArkTS 方法调用栈与耗时，热点函数一目了然，绿色"ArkTS"标记的行双击可跳源码。官方案例中的典型读法：某卡顿帧内 `initialRenderView` 占 52.7%、`__lazyForEachItemGenFunction` 占 22.9%（第 7.2 节）。注意 program 节点表示程序进入纯 Native 执行段，这里看不到 ArkTS 信息，需切到 Callstack（Native）泳道。

**ArkUI Component**：自定义组件与系统组件的创建、布局、渲染次数与耗时。用法是在 Details 区过滤目标组件，对比它与其他组件的耗时量级；耗时异常高的组件，配合 Frame Layout 或 ArkUI Inspector 看嵌套结构。

**Anomaly**：系统自动告警泳道，目前两类（出处：《Frame 分析》）：

- 主线程图片解码耗时超过 VSync 周期 50% 时红色告警（"Image decoding has exceeded 50% of the VSync time"），详情区给出图片名、次数、总/平均耗时、源尺寸与目标尺寸。
- Worker/TaskPool 跨线程对象传递的序列化、反序列化耗时超阈值告警，默认阈值 8ms，可在泳道配置里改。

限制：由于隐私安全政策，已上架应用市场的应用不支持录制 Anomaly 泳道。

**User Events**：记录用户事件的开始时间（Input Time）、应用开始处理时间（Processing Start）与处理耗时。做响应时延分析时，Input Time 到 Processing Start 的间隔直接回答"事件在队列里排了多久"。

## 可变帧率监测：Display Vsync 与 DisplaySync_cb

高刷设备的帧率是动态的，Frame 泳道提供两条专用子泳道（出处：《Frame 分析》）：

- **Display Vsync**：显示时间段内的屏幕刷新率，框选后按 <=30Hz、30~60、60~90、>90Hz 四档做分布统计，给出各档占比、最小/最大/平均时长和平均 Hz。用于验证"该降帧的场景降了没有"。仅支持配备硬件屏幕的设备。
- **DisplaySync_cb(tid)**：显示 DisplaySync、XComponent 两类接口组件对应的帧率与渲染时间。官方提醒：DisplaySync 的渲染在 UI 主线程执行，多个组件同时渲染时只能排队，任何一个组件延迟都会挤占其他组件的渲染时间——泳道上会把可能的掉帧标红。

**Animation / Component Animation 泳道**：Animation 子泳道给出每个动效的响应时延、持续时间、完成时延、期望帧率与 FPS，并按阈值着色（响应时延 <=85ms 绿、85~150 浅绿、150~250 浅红、>250 深红；FPS 达到期望帧率的 91.7% 为绿）。Component Animation 子泳道（DevEco Studio 6.0.0.828 起）从组件维度列出属性动画、显式动画、关键帧动画、页面转场的起止时间、帧率、曲线类型与影响的属性。调转场和动效问题时先看这里，第 8.2 节的 H:Animator 分析由此展开。

## 实战示例：从红帧到代码行

把前面所有功能串成一次标准操作（即第 7.2 节案例的工具视角）：

1. 录制 Frame（勾选 ArkUI Component），复现滑动卡顿。
2. 框选卡顿区段 → Statistics 看丢帧率 → Frame List 按 Jank Type 排序，确认 AppDeadlineMissed。
3. 点选一个红帧 → 详情看 Expected Duration vs 实际耗时 → 跳转关联切片。
4. ArkTS Callstack 找热点函数 → 双击跳源码。
5. ArkUI Component 过滤耗时组件 → Open Layout 看嵌套。
6. 修复后重录，Statistics 对比丢帧率。

常见坑：忘了勾 ArkUI Component；拿上架版本录 Anomaly（录不到）；把首帧红当问题追半天（首帧重建整树，红是常态，见第 8.1 节）；框选区段太大导致统计被稀释——分析单点问题时框 2~3 秒足够。

## 参考资料

- 《Frame 分析》（ide-insight-session-frame）
- 《性能问题定位：深度录制》（deep-recording）
- 《帧率问题分析》（bpta-zhenlv）
- 《点击完成时延分析》（bpta-click-to-complete-delay-analysis）
- 《ArkUI 分析》（ide-arkui-analysis）
- 《基础耗时：Time 分析》（ide-insight-session-time）
