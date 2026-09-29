# 9.2 AppFreeze 日志分析与定位

## 先拿到日志，再谈分析

AppFreeze 日志由 FaultLogger 模块管理，与崩溃日志放在一起，三种获取方式（出处：《AppFreeze（应用冻屏）检测》）：

1. **DevEco Studio**：自动收集设备 `/data/log/faultlog/faultlogger/` 下的日志，在 FaultLog 窗口按"包名 > 故障类型 > 故障时间"分类展示，关键信息高亮，堆栈可直接跳转代码。
2. **HiAppEvent 订阅**：应用内订阅 APP_FREEZE 事件，通过事件参数的 `external_log` 字段读取故障日志文件内容。线上场景的标配。
3. **hdc 命令**（需打开开发者选项）：

```shell
hdc file recv /data/log/faultlog/faultlogger 本地目录
```

故障日志文件名格式为 `appfreeze-进程名-进程UID-毫秒级时间.log`。API 21 起的增强日志（采样栈）单独存放在 `/data/log/faultlog/freeze_ext/`，文件名格式为 `freeze-cpuinfo-ext-进程名-进程UID-毫秒级时间`。

分析原则：AppFreeze 日志要和 hilog 流水日志配合看。日志里的 TIMESTAMP 字段给出故障时刻，用它减去超时阈值（6s 或 8s），得到的区间就是要在 hilog 里重点翻阅的时间窗。

## 日志地图：六个模块

一份 AppFreeze 日志从头到尾固定分六块。先建立整体地图，再逐块精读。

[图：AppFreeze 日志结构示意图。自上而下六个区块：①日志头部（设备/应用/Reason/页面切换轨迹）②MSG 与 EventHandler dump ③故障进程各线程堆栈 ④BinderCatcher 与 PeerBinder 对端信息 ⑤CPU 使用率 ⑥整机内存信息。]

### 模块一：头部信息

```text
Reason:THREAD_BLOCK_6S
appfreeze: com.samples.freezedebug THREAD_BLOCK_6S at 20250628140837
Foreground:Yes
Pid:13680
Process life time:18s
HitraceIdInfo: hitrace_id: a92ab27238f409a, span_id: 1cd61c9, ...
Page switch history:
  14:08:30:327 /ets/pages/Index:Appfreeze
  14:08:28:986 /ets/pages/Index
  14:08:26:502 :enters foreground
```

重点看四个字段：

- **Reason**：故障类型，THREAD_BLOCK_6S / APP_INPUT_BLOCK / LIFECYCLE_TIMEOUT，决定整份日志的读法。
- **Page switch history**（API 20 起）：故障前的页面切换轨迹。用户从哪个页面、几点几分切过来，操作路径一目了然，对复现帮助极大。
- **HitraceIdInfo**（API 20 起，THREAD_BLOCK_6S 时输出）：HiTraceChain 的跟踪标识，用它可以在 hilog 里把这次故障相关的调用链日志串出来。
- **NOTE 行**（API 20 起）：如果整机处于低内存或热限频状态，系统会输出 NOTE 提示"当前故障可能由系统资源告警引起，可忽略"。看到这行先别急着改代码。

### 模块二：MSG 与 EventHandler dump

THREAD_BLOCK 的核心证据区：

```text
MSG =
Fault time:2025/06/28-14:08:34
App main thread is not response!
Main handler dump start time: 2025-06-28 14:08:34.067
 EventHandler dump begin curTime: 2025-06-28 14:08:34.067
 Event runner (Thread name = , Thread ID = 13680) is running
 Current Running: start at 2025-06-28 14:08:27.354, Event { send time = 2025-06-28 14:08:22.353,
   trigger time = 2025-06-28 14:08:27.354, task name = uv_timer_task, caller = [ohos_loop_handler.cpp(OnTriggered:72)] }
 History event queue information: ...
 VIP priority event queue information:
 No.1 : Event { ..., caller = [watchdog.cpp(Timer:233)] }
 No.2 : Event { ..., caller = [watchdog.cpp(Timer:233)] }
 Total size of VIP events : 2
 Total event size : 2
```

官方建议只重点关注三个时间（出处：《AppFreeze（应用冻屏）检测》）：EventHandler dump begin curTime、各任务的 trigger time 和 completeTime time。读法：

- **Current Running** 是故障时主线程正在执行的任务。上面这份日志里，它是一个 `uv_timer_task`（定时器回调），从 14:08:27 开始跑，到 dump 时刻（14:08:34）还没跑完——单个任务跑了 7 秒，就是它堵住了主线程。
- **VIP 队列**里那两条 `watchdog.cpp(Timer)` 就是判活任务，它们被投递了但一直在排队，这正是 THREAD_BLOCK 被判定的直接证据。
- **各优先级队列的 Total size**：如果队列里积着几十上百条任务，而 Current Running 都是短任务，那就是"主线程繁忙"形态，不是单个长任务。

### 模块三：故障进程堆栈

每个线程一段，主线程在最前面。THREAD_BLOCK 的主线程堆栈通常能直接指出元凶：

```text
Tid:13680, Name:les.freezedebug
state=S, utime=0, stime=0, priority=0, nice=-20, clk=100
#00 pc ... [shmm](__kernel_gettimeofday+72)
#01 pc ... (gettimeofday+40)
#02 pc ... libark_jsruntime.so(BuiltinsDate::Now...)
#05 at wait15s (entry/src/main/ets/pages/Index.ets:16:10)
#09 pc ... libtimer.z.so(Timer::TimerCallback...)
```

这份示例栈读出来：定时器回调里执行了 `wait15s` 函数（Index.ets 第 16 行），函数内部在反复调用 `Date.now`——一个典型的"主线程里忙等"写法。结合模块二里 Current Running 的 `uv_timer_task`，证据链闭合。

API 23 起线程号下新增 `state=S, utime=0, stime=0, ...` 行，是另一个重要判据：对比 THREAD_BLOCK_3S 和 THREAD_BLOCK_6S 两份堆栈的 utime/stime，如果毫无变化，说明这 3 秒里主线程根本没被调度——业务代码又没有阻塞调用的话，就可以怀疑系统调度问题，而不是应用代码问题。

堆栈缺失时日志会打印替代信息，各自有明确含义：`has been crashed`（目标进程已崩溃，去看同时刻的 crash 日志）、`SIGDUMP error`（收到抓栈请求时已退出）、`is dumping`（正在 dump，短时间内重复请求）、`State: S`（Sleep 中）、`errno(2)`（无响应信号，超时 1s 未返回，改打内核栈和 /proc/status）。

### 模块四：BinderCatcher 与 PeerBinder

主线程堆栈栈顶是 binder 调用时，这一块就是主战场：

```text
PeerBinderCatcher -- pid==13680
BinderCatcher --
    1733:2285 to 3712:3712 code b wait:1.365925521 s frz_state:3, ...
pid context   request started max ready free_async_space
13680 binder    0     2      16  3     520192
```

- 每行的格式是"客户端进程:线程 to 服务端进程:线程 code 业务码 wait 等待时长"。wait 超过秒级的同步调用都值得追查——顺着对端 pid 找到对端进程，看它为什么慢。
- 下方的进程 binder 线程表给出每个进程的 IPC 线程状况：started（已启动线程数）、max（上限 16）、ready（空闲数）。如果对端进程 ready=0 且 started=max，就是官方根因分类里的"对端 Binder 线程满"——对端所有 IPC 线程都被占住，我们的同步调用只能排队。解法：改用异步接口，或者降低调用频率。
- **PeerBinder Stacktrace**：对端进程也有卡死迹象时，系统会直接抓对端的堆栈附在日志里。主线程等 IPC、对端在干什么，两段堆栈对着看，证据链就完整了。

### 模块五与六：CPU 与内存

```text
Load average: 14.3 / 12.9 / 11.4
Total: 22.45%; User Space: 13.64%; Kernel Space: 8.81%; iowait: 0.33%; idle: 77.15%
Details of Processes:
    PID   Total Usage    Name
    13680    8.86%       com.samples.freezedebug
```

CPU 段用于回答"是不是整机太忙"：Load average 持续走高、故障进程自身 CPU 占比却很低，说明主线程在等资源而不是在跑代码。内存段的 `ReclaimAvailBuffer`（整机剩余可用内存）用于确认低内存背景，与头部 NOTE 行互相印证。

## 三种典型形态的定位路径

把上面的模块串起来，三种最高频的形态各有固定打法：

**形态 A：主线程跑耗时任务。** 证据：Current Running 是业务任务且持续时间长；堆栈栈顶落在业务函数（或经 libark_jsruntime 落到 ets 行号）。行动：release 包用 hstack 配 Source Map 反解堆栈，debug 包直接 IDE 跳转；把耗时逻辑挪到 TaskPool/Worker，或拆分成分片任务。

**形态 B：等锁/死锁。** 证据：堆栈栈顶是锁等待（futex、mutex lock 类符号）；3s 与 6s 两份堆栈 utime 不变；其他线程堆栈里能找到锁的持有者。行动：找出持锁线程在干什么，检查锁粒度和锁顺序。增强日志的多次采样栈对这种形态特别有效——同一堆栈在多次采样中原地不动，就是死等。

**形态 C：IPC 等待。** 证据：堆栈栈顶是 binder 调用；BinderCatcher 中对应行 wait 时长秒级；对端 ready=0 或对端堆栈显示对端也卡住。行动：评估该调用能否异步化；对端是系统服务且反复出现时，保留日志向系统侧提单。

LIFECYCLE_TIMEOUT 单独说：它的 MSG 段不是 EventHandler dump，而是 server/client 两侧带时间戳的生命周期动作时间线（`JsUIAbility::OnStart begin/end`、`OnSceneCreated begin/end`……）。相邻记录的时间差就是每步耗时，找到断档的那一步——begin 出现、end 缺席的位置，就是卡住的生命周期回调，官方文档为每个动作标注了"失败原因"和"是否需要应用处理"。

## 增强日志：解决"栈顶不是真凶"

普通日志只抓一份堆栈，有个固有缺陷：主线程繁忙时抓栈有延迟，栈顶落在哪纯看运气，经常抓到的不是真正的耗时点（瞬时栈问题）。从 API 21 起，系统在 THREAD_BLOCK_3S / LIFECYCLE_HALF_TIMEOUT 触发时就开始周期采集主线程调用栈，到 6s 故障时停止，一般抓 1~10 份，这就是增强日志（出处：《AppFreeze（应用冻屏）检测》）。

增强日志比原生日志多两样东西：

- **多份采样栈 + SubmitterStacktrace**：多次采样中反复出现的栈帧就是热点；每份栈还带任务提交者调用栈（最多 16 层），回答"谁把这个任务塞进队列的"——主线程堆栈只能看到任务在执行，提交者栈才能看到任务从哪来。
- **CPU 时间统计**：`StaticsDuration = CpuTime + SyncWaitTime` 的分解直接给出定性结论。CpuTime 大说明主线程真在算（形态 A）；SyncWaitTime 大说明主线程在等（形态 B/C）。`SupplyAvailableTime`（调度可优化时间）小，说明主线程繁忙是应用自身任务太多，别指望系统调度救场。

从 HarmonyOS 6.1.0.125 起，增强日志默认合并到 APP_FREEZE 事件 external_log 指向文件的结尾；需要单独文件时在 app.json5 配置 `DFX_APPFREEZE_LOG_OPTIONS` 的 `mainthread_sampling:enable`。

## 举一反三

这份读日志的方法论可以迁移到所有 FaultLogger 产出的日志：jscrash、cppcrash（第 15 章）与 appfreeze 共享头部格式、堆栈格式和聚类方法。官方也建议按堆栈做聚类——不同版本、不同时间产生的故障日志，大部分字段都变，只有故障线程调用栈稳定不变，相同栈即相同根因。线上场景把 external_log 回收后按栈聚类统计，能把"几百份日志"收敛成"几个问题"，这是 AppFreeze 治理从单点分析走向运营的关键一步。

## 参考资料

- 《AppFreeze（应用冻屏）检测》（appfreeze-guidelines）
- 《应用冻屏故障检测机制及规格说明》（bpta-stability-appfreeze-fault-detection-mechanism）
- 《应用冻屏问题定位方法》（bpta-stability-app-freeze-way）
- 《开发阶段应用冻屏问题定位》（bpta-app-freeze-in-develop）
- 《应用冻屏事件介绍》（hiappevent-watcher-freeze-events）
- 《堆栈解析工具（hstack）》（ide-command-line-hstack）
