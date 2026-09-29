# 第 14 章：DevEco Profiler

第 13 章讲了命令行路径的 hitrace 和 SmartPerf，本章讲 IDE 路径的主力工具：DevEco Studio 内置的 Profiler。两者的底层数据是同源的——都是系统打点加上采样——但 Profiler 在数据之上做了场景化封装：把几十条泳道按问题类型组织成"会话模板"，再叠上 ArkTS 函数调用栈、组件级耗时、内存对象视图这些应用层信息。对应用开发者来说，Profiler 是性能分析的第一入口，命令行工具是补充。

Profiler 的使用模型统一：选择设备与应用进程 → 选择模板创建会话（Create Session）→ 录制（支持设置录制时长与泳道开关）→ 停止后解析 → 在时间线上框选分析。历史数据可以导出保存，也能通过会话区的 Open File 导入重新分析——包括.hittrace、.heapsnapshot 等格式。本章讲三个最常用的会话：

**14.1 Frame 会话：丢帧与帧耗时分析。** 这是全书出现频率最高的工具。RS Frame / App Frame 泳道的红绿帧判读、期望开始/结束时间线、Jank Type 三种取值、Statistics 与 Frame List 统计、ArkTS Callstack 热点函数、ArkUI Component 组件耗时（默认关闭需手动勾选）、Display Vsync 可变帧率监测、Anomaly 泳道的图片解码与序列化告警。第 7、8 章的案例全部建立在这节的能力上。

**14.2 Launch 会话：启动性能分析。** Launch 模板把冷启动拆解为 Process Creating、Application Launching 等阶段，配合"冷启动首帧完成时延"这条核心指标线。官方案例中预期 600ms 的启动实测超过 800ms，问题定位到 Ability 生命周期里的耗时操作——这一节讲怎么找到这条时延的起止点、怎么用 `H:FlushMessages > H:SendCommands > H:MarshRSTransactionData` 三个短 Trace 定位应用通知 RS 渲染的时刻。

**14.3 CPU / Memory / Snapshot 会话。** CPU 会话的采样与调用栈视图（含 View Integrated Scheduling Chain 唤醒链开关）；Memory 会话的内存曲线；Snapshot 的堆快照对比（Comparison/Containment/Statistics 三视图）；Allocation 模板的泄漏检测（直接给出泄漏对象的分配调用栈）；以及 ArkWeb 场景的 Sub Resource 资源分析。

读本章前如果还没做过一次完整录制，建议先按 14.1 节的步骤对自己应用录一段 Frame——工具章节的读法永远是边读边开 IDE。Profiler 各泳道采集的是同一段时间的数据，会话之间不是非此即彼：一个复杂问题常常要 Frame 看帧、CPU 看调度、Snapshot 看内存，交叉确认。
