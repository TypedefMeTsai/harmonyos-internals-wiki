# 24.1 线上质量：崩溃、无响应与性能采集

## hiAppEvent：系统替你收集的故障现场

应用自己 try-catch 能拿到 JS 异常，但拿不到系统视角的故障全貌。hiAppEvent 的价值在于：系统服务在故障发生瞬间已经抓好了现场，应用通过订阅拿到结构化数据。

### 订阅什么

通过 `hiAppEvent.addWatcher` 订阅系统事件（domain 为 `hiAppEvent.domain.OS`），与质量直接相关的事件：

- **APP_CRASH（崩溃事件）**：字段含 crash_type（崩溃类型）、foreground（前后台状态）、release_type、cpu_abi、bundle_version、pid/uid、exception（异常类型、原因、调用栈）、hilog（发生时日志）、memory（内存信息）、process_life_time（故障进程存活时间）、external_log（崩溃日志文件路径）、page_switch_log（页面切换日志）。
- **APP_FREEZE（应用冻屏事件）**：字段含 exception、hilog、event_handler（主线程未处理消息，另有 event_handler_size_3s/6s 统计）、peer_binder（同步 binder 调用信息）、threads（全量线程调用栈）、memory、external_log、page_switch_log。这套字段就是第 9 章 AppFreeze 分析的线上版——拿到 threads 和 peer_binder，THREAD_BLOCK 是锁等待还是 binder 等待，不下设备也能判断。
- **appFreezeWarning（冻屏告警）**：冻屏发生前的预警事件，字段类似。适合做趋势监控：warning 上涨但 freeze 没涨，说明应用在往悬崖边走。
- **启动耗时事件**：系统对启动过程打点的结果，含点击时间与动效结束时间，订阅后每次启动都能拿到耗时数据（注意：首帧绘制前终止的启动不生成该事件）。

事件订阅的规格：普通应用、应用分身、元服务、输入法应用（API 22+）都可订阅崩溃与冻屏事件。崩溃和冻屏事件还支持 `setEventParam` 塞自定义参数（如当前页面、业务版本），让平台数据带上业务上下文。

### 订阅的工程细节

- 订阅代码放在 Ability 的 onCreate 里，尽早注册。
- 崩溃发生在进程死亡时，onReceive 回调在**下次启动**时才能把事件交给你——上报逻辑要容忍"T+1 次启动才上报本次崩溃"。官方 FAQ 的完整示例演示了崩溃后读取 external_log 文件的姿势（崩溃日志文件可读，验证权限后可上传后删除）。
- 崩溃日志体量可以管控：`hiAppEvent.configEventPolicy` 的 appCrashPolicy 支持页面切换日志开关、日志截断大小（logFileCutoffSzBytes）、精简 maps 打印、native 崩溃 minidump 等（部分策略项有 API 版本要求，用 canIUse 判断）。
- 别在 onReceive 里做重活（网络上传放队列异步做），它是事件分发回调，不是后台任务。

## 性能打点：HiTraceMeter 与启动耗时

线上性能数据的两条路：

**系统打点**：启动耗时事件（上文）是现成的，订阅即可，覆盖点击到动效结束。这是成本最低、口径最统一的启动质量数据源，官方建议通过大数据分析这些上报判断启动耗时是否健康（《启动时延检测》最佳实践）。

**自定义打点**：HiTraceMeter 的 startTrace/finishTrace 不止在本地 hitrace 里可见，也可以作为业务耗时的度量点。打点原则：

- 打在关键路径的"段"上：冷启动各阶段、核心页面加载、核心交互（下单、发布）的完成时延；
- 单条 trace 记录总长度限制 512 字节，name + customCategory + customArgs 长度之和建议不超过 420 字节（官方参考文档建议）；
- 打点有开销，release 版本只保留关键段，调试级打点用构建开关关掉；
- `hidebug.startAppTraceCapture` 可按 tag 范围做自动化 trace 采集，适合灰度环境的定向抓包，官方建议先用 hitrace 命令筛出关键范围再配进接口（采集性能消耗与范围正相关）。

## 平台侧：AGC 质量服务

上架应用的现成看板在 AppGallery Connect：

- **应用质量管理（APMS）**：崩溃事件聚类与日志、异常管理、告警规则配置（监控时段、频率、触发条件）。官方崩溃最佳实践给出的运维流程：配告警 → 筛选问题（发生次数、影响设备数）→ TOP 问题堆栈分析 → 修复 → 验证。
- **启动和卡顿分析报表**：问题趋势、问题排序、问题列表三部分，指标含"冻结帧异常率"（单位时间 1000h 内发生冻结帧的次数）。
- **云测试性能检测**：上架前用云端真机跑性能检测，启动时延、帧率等指标按《应用性能体验建议》的规则判定。

自建体系和 AGC 不是二选一：AGC 看大盘（聚类、趋势、告警），hiAppEvent 订阅拿原始事件按业务维度细分（哪个页面、哪个 A/B 分组），两边数据口径对齐后互相印证。

## 发布质量门禁：把指标写进流程

线上体系建好后，最后一步是让它挡住坏版本。一套可执行的门禁：

**灰度期指标**（hiAppEvent 上报 + AGC 报表）：

- 崩溃率：单位活跃用户崩溃次数，超基线即阻断放量；
- AppFreeze 率：同上，另看 appFreezeWarning 的趋势先行指标；
- 启动耗时分布：P50/P90 与上一版本对比，P90 劣化超过阈值（比如 10%）即排查；
- 冻结帧异常率：AGC 报表直接给。

**版本对比纪律**：每个版本冻结一份基线数据；新版本灰度数据只和基线比，不和"感觉"比；指标全部达标才放量，任何一项劣化先回查变更集——性能门禁失效的头号原因是"这次先放，下个版本再修"。

**归因配合**：门禁报警后按本书工具链回溯——崩溃/APMS 堆栈 → FaultLog 分析（第 15 章）；启动劣化 → 启动耗时事件分段 + 灰度设备上 aa start -W 复测；卡顿劣化 → 冻结帧 TOP 场景 + 本地 Frame 会话复现。线上数据负责"发现问题、圈定范围"，本地工具负责"看清根因"，这个分工不要混。

## 参考资料

- 官方文档：《HiAppEvent 介绍》（hiappevent-intro、event-subscription-overview）
- 官方文档：《订阅崩溃事件（ArkTS）》（hiappevent-watcher-crash-events-arkts）
- 官方文档：《订阅应用冻屏事件》（hiappevent-watcher-freeze-events-arkts）、《冻屏告警》（hiappevent-watcher-appfreezewarning-events-arkts）
- 官方文档：《订阅启动耗时事件》（hiappevent-watcher-app-launch-event）
- 官方文档：《崩溃监控最佳实践》（bpta-crash-monitor-practice）、《运维态崩溃分析》（bpta-app-crash-in-operation）
- 官方文档：《启动时延检测》（bpta-performance-startup-time-detection）
- 官方文档：AGC 帮助《质量报告/启动和卡顿分析》（agc-help-quality-report）
- 官方文档：《hiAppEvent 配置崩溃事件策略》（hiappevent-watcher-crash-events，configEventPolicy）
