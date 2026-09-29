# 第 13 章：HiTrace 与 SmartPerf

从本章开始进入第三部分：工具与方法论。第二部分的每一章都在用 Trace，但都是"用到哪个讲哪个"；这一部分把工具本身讲透。第一个要讲的就是 HiTrace——鸿蒙性能数据的事实标准。

HiTrace 之于鸿蒙，相当于 Perfetto/atrace 之于 Android。系统各模块（ArkUI、图形、窗口、调度、Binder、输入）在关键路径上预埋了大量打点，应用也可以用 HiTraceMeter 接口打自己的点。`hitrace` 命令行工具负责把这些打点从内核缓冲区里抓出来，输出成文本或二进制 trace 文件。DevEco Profiler 的底层数据采集同样走这条路——理解 hitrace，就理解了 Profiler 看到的一切是从哪来的。

抓到的 trace 文件需要可视化工具才能变成泳道图。开源的 SmartPerf-Host（OpenHarmony 官方发行版）是主力：把 hitrace 的 .sys/.ftrace 文件拖进去，就能看到 CPU 调度、频点、线程时间片、帧率、各进程打点的完整时间线，也可以用 Perfetto UI 打开二进制 trace。与 SmartPerf-Host 配套的还有设备端的 SP_daemon（SmartPerf 采集命令），它不抓打点，而是周期性采样帧率、CPU、GPU、内存、电量、温度、流量这些"指标型"数据，输出 CSV，适合做场景级的定量对比。

本章三节：

**13.1 HiTrace：命令与 tag 体系。** `hitrace` 的完整用法——三种采集模式（定时、快照、录制）、缓冲区与文件大小参数、70 多个 tag 的分工、文本与二进制两种输出格式、以及四个最高频的报错（错误码 1、1004、illegal path、not support category）的原因和解法。

**13.2 Frame 模板与关键 Trace 点速读。** 一张速读表：渲染链路上二十来个关键 Trace 点，每个标注"正常什么样、异常什么样、指向什么问题"。这是本书第二部分所有案例的索引表，也是日常翻 Trace 时最值得放在手边的一页。

**13.3 SmartPerf 与 SmartPerf-Host。** SP_daemon 的采集命令（-PKG/-PID、帧率、CPU、GPU、内存、功耗、流量、响应/完成时延的场景化采集），以及 SmartPerf-Host 的基本操作与五个内置分析模板。

建议的读法：13.1 当手册，用到再查；13.2 精读，它是读 Trace 的字典；13.3 做一次实操，抓一份自己应用的帧率和功耗数据。
