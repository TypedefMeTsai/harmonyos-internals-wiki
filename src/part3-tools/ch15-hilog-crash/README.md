# 第 15 章：HiLog 与崩溃分析

Trace 回答"性能怎么花的"，日志回答"系统刚刚经历了什么"。鸿蒙的日志体系分两半：一半是 hilog——持续滚动的流水日志，应用和系统模块往里写运行状态，开发者用命令实时翻阅；另一半是 FaultLog——故障档案，崩溃（JsCrash/CppCrash）、应用冻屏（AppFreeze）这类事故发生时，由 FaultLogger 模块抓取现场，生成一份结构化日志存档。前者是监控录像，后者是事故报告，排查线上问题两个都要会读。

hilog 之于鸿蒙相当于 logcat 之于 Android，但有两个差异要先知道。其一，hilog 有严格的流量治理：每个进程、每个 domain 每秒的日志量超限会被直接丢弃并打印超限提示，官方建议每个应用进程每秒日志量不超过 50KB——日志写多了不但看不到，还会丢关键的。其二，商用设备分 log 版本和 nolog 版本，nolog 版本默认不打印日志，开启开发者模式后全局级别也只有 WARN，查问题前要先确认设备的日志级别设置，否则"什么日志都没有"可能只是被级别挡住了。

本章两节：

**15.1 hilog 体系与高效查日志。** domain/tag/level 三层结构、日志行格式（A03200 这样的前缀怎么读）、`hdc shell hilog` 的过滤组合（按级别、domain、tag、pid、正则）、缓冲区管理与落盘任务、超限机制三种丢日志形态（LOGLIMIT / Slow reader missed / write socket failed）的识别与处置。

**15.2 FaultLog、CppCrash 与应用事件。** 故障日志的存放位置与获取方式（DevEco Studio FaultLog 窗口、hdc file recv、HiAppEvent 订阅）；CppCrash 日志的结构化解读——Reason 信号含义、寄存器与 Memory near registers、堆栈回溯、hstack 反混淆；应用侧的两条可观测通道：errorManager 监听未捕获异常、hiAppEvent 做故障订阅与自定义打点。

第 9 章的 AppFreeze 日志也属于 FaultLog 体系，格式与获取方式通用，本章不再重复，重点放在崩溃类日志。
