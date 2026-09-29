# 13.3 SmartPerf 与 SmartPerf-Host

## 这个工具解决什么问题

hitrace 抓的是"事件"——每个打点何时发生、持续多久，回答"为什么"。但实际工作里还有一类问题是事件流回答不了的："这次优化到底省了多少电""帧率抖动是多少""这个场景下 CPU 占了几个百分点"。这类问题需要的是周期采样的指标数据：每秒一次，记下当前的帧率、CPU 占用、内存、电量、温度。

SmartPerf 就是干这个的。它分两端：设备端的 `SP_daemon` 命令负责采集，输出 CSV 或日志；PC 端的 SmartPerf-Host 负责可视化和深度分析，同时还能直接打开 hitrace 抓的 trace 文件。两者合起来覆盖了"指标采样 + 事件时间线"两类需求（出处：《SmartPerf 使用指导》《性能 FAQ-45》）。

## SP_daemon 基本使用

SP_daemon 通过 hdc shell 在设备上执行。先看参数全景（官方 help 输出整理）：

| 参数 | 作用 |
| --- | --- |
| `-N n` | 采集次数（默认 0 表示持续），如 `-N 10` |
| `-PKG 包名` | 指定应用包名（采集应用级数据必填） |
| `-PID pid` | 指定进程号 |
| `-c` | 整机 CPU 频率/使用率、进程 CPU 占用与负载 |
| `-ci` | CPU 指令数与周期数 |
| `-g` | GPU 频率与负载 |
| `-f` | 应用帧率 fps、jitter 与刷新率 |
| `-profilerfps` | 帧率与时间戳（配 `-sections` 分段） |
| `-t` | 剩余电量与温度 |
| `-p` | 功耗与电压（部分设备不支持） |
| `-r` | 进程内存与整机内存 |
| `-net` | 上下行流量 |
| `-d` | DDR 信息 |
| `-snapshot` | 屏幕截图 |
| `-threads` | 线程信息（须配 -PID 或 -PKG） |
| `-start` / `-stop` | 手动起停采集 |
| `-OUT` | CSV 输出路径 |
| `-recordcapacity` | 采集前后电量差 |
| `-screen` / `-deviceinfo` | 屏幕分辨率 / 设备信息（单独使用） |
| `-editor responseTime/completeTime 包名` | 场景化采集页面响应/完成时延 |
| `-server` / `-clear` | 常驻服务模式 / 清理 |

典型用法（官方示例）：

```shell
# 整机级：采 20 次 CPU、GPU、电量、功耗、内存、流量、DDR
SP_daemon -N 20 -c -g -t -p -r -net -d

# 应用级：对 ohos.samples.ecg 采 20 次，加帧率
SP_daemon -N 20 -PKG ohos.samples.ecg -c -g -t -p -f -r -net

# 手动起停：先 start，操作场景，再 stop
SP_daemon -start -c
SP_daemon -stop

# 场景化时延采集
SP_daemon -editor responseTime ohos.samples.ecg
SP_daemon -editor completeTime ohos.samples.ecg
```

输出为周期性采样行，配 `-OUT` 落 CSV 后可直接进表格工具画图。

**实战建议**：

- 做优化前后的 A/B 对比时，两次采集的参数、场景操作、设备状态（电量区间、温度、网络）尽量一致，指标型数据对环境敏感。
- `-p` 功耗采集在部分设备上不支持，先用 `-t`（电量）加 `-recordcapacity` 做场景电量差兜底。
- 帧率相关参数有三档：`-f`（fps 与 jitter）、`-profilerfps`（带时间戳，可配 `-sections` 分段）、`-ohtestfps`（给自动化测试工具用）。手工分析用 `-f`，要逐帧时间戳用 `-profilerfps`。
- `-N 0` 持续采集时记得 stop，长期挂着会持续产生开销。

## SmartPerf-Host：时间线分析

SmartPerf-Host 是 OpenHarmony 官方发行的 PC 端可视化工具（gitcode.com/openharmony/developtools_smartperf_host，releases 页面下载，Windows/Mac/Linux 均可）。它是一款"深入挖掘和细粒度展示数据的性能功耗调优工具"：采集 CPU 调度、频点、进程线程时间片、堆内存、帧率等数据，以泳道图呈现（出处：《性能 FAQ-45》）。

**打开 trace**：直接把 hitrace 抓的 .sys 文件（或 DevEco Profiler 导出的数据）拖进窗口，等待解析后呈现完整时间线。界面结构与主流 trace 查看器一致：上方时间轴，左侧泳道列表，框选时间段后下方出统计详情。几个高频操作：

- **框选 + 统计**：在时间轴上拖选一段，下方统计区给出该区间内各泳道的聚合数据（如某线程 Running 总时长、某频点占比）。
- **线程状态判读**：CPU 时间片泳道上，Running（绿）、Runnable（可运行排队）、Sleeping 三色是判断"线程在干活还是在等"的基础，第 7.1 节"排除系统侧"和第 8.2 节"唤醒链回溯"都依赖它。
- **频点对照**：Frequency 泳道与线程时间片对照，确认关键线程是否跑在大核高频上。

**五个内置分析模板**（出处：《性能 FAQ-45》）：帧率分析、CPU/线程调度分析、应用启动分析、TaskPool 分析、动效分析。模板是预置的泳道组合与统计视图，例如帧率分析模板自动聚合帧周期与丢帧统计，TaskPool 模板聚焦任务线程的排队与执行。没有现成模板覆盖的场景，用自由泳道视图手动组合。

**与 DevEco Profiler 的分工**：这是实际使用中最常见的困惑。原则是——Profiler 优先，SmartPerf-Host 兜底。DevEco Profiler 的会话模板（Frame/Launch/CPU/Memory）对应用场景做了封装，带 ArkTS Callstack、ArkUI Component 这类应用层视图，日常应用分析都在 Profiler 完成；SmartPerf-Host 的优势在系统级视角（全系统进程的调度、频点、负载一览）和对裸 hitrace 文件的兼容。抓来的 .sys 文件两头都能开，哪边顺手用哪边。

## 常见问题与坑

**SP_daemon 报权限或找不到进程**：应用级采集要求目标进程在运行且包名正确，先 `pidof 包名` 确认；部分参数（如 -p）依赖设备硬件支持，不支持就是真不支持，换 -t。

**SmartPerf-Host 打不开 trace 文件**：先确认文件完整性——hdc file recv 大文件可能传输中断，比对大小；录制模式产生多个分卷文件时要全部拉取。文本格式 .ftrace 与二进制 .sys 都支持，但混合格式需分别打开。

**采样数据与 trace 对不上**：SP_daemon 的周期采样（1s 级）和 hitrace 的事件流（微秒级）精度差着几个数量级，两者回答不同层次的问题。先说"功耗涨了 5%"（SP_daemon），再问"涨在哪"（hitrace），顺序别反——拿 1 秒粒度的采样去解释毫秒级的帧超时，一定会得出错误结论。

## 参考资料

- 《SmartPerf 使用指导》（smartperf-guidelines）
- 《性能 FAQ-45：SmartPerf-Host 模板》（faqs-performance-45）
- 《hitrace》（hitrace）
- 《HiSmartPerf / SmartPerf-Host》（developtools_smartperf_host 发行版说明）
- 《帧率问题分析》（bpta-zhenlv）
