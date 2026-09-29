# 9.1 AppFreeze 判定机制与故障类型

## 为什么系统要主动杀死"卡住"的应用

应用主线程是一个事件循环：输入事件、Vsync 刷新、生命周期回调、定时器，所有任务都排队等它处理。主线程一旦长时间不消费队列，应用对外的一切表现都会停止——不响应触摸、不刷新画面、不回应系统的调度指令。从用户角度看应用"死了"，从系统角度看进程还活着，占着内存和资源。

这种半死不活的状态比崩溃更糟：崩溃至少干脆，卡死会让用户对着一块冻住的屏幕反复点击。所以系统必须替用户做这个决定：检测到应用失去响应能力，抓取现场信息，然后强制终止进程，让应用回到可恢复的初始状态。这就是 AppFreeze 检测存在的理由——它不是惩罚，是兜底。

## 三个维度的检测

应用冻屏检测从主线程任务队列、用户输入事件、UIAbility 生命周期三个维度出发，各自设定时间阈值做超时判定（出处：《应用冻屏故障检测机制及规格说明》）。一旦触发超时，进入故障处理流程：收集信息、生成事件、上报、分发给应用内已注册的订阅者，同时强制终止应用进程。

### THREAD_BLOCK：主线程卡死

检测原理：应用的 watchdog 线程定期向主线程插入判活检测任务。判活任务超过 3s 未被执行，上报 THREAD_BLOCK_3S 告警事件；超过 6s 仍未执行，上报 THREAD_BLOCK_6S 主线程卡死事件。两个事件匹配后生成完整的 THREAD_BLOCK 应用无响应日志，并强制终止应用（出处：《AppFreeze（应用冻屏）检测》《应用冻屏场景化最佳实践》）。

[图：watchdog 判活机制时序图。watchdog 线程周期性地向主线程事件队列插入判活任务；正常时任务被及时消费；主线程被某个长任务占住时，判活任务排队等待，3s 触发告警、6s 触发故障。]

THREAD_BLOCK_3S 是 API 26 起新增的告警级别事件，只告警不杀进程，给开发者一个提前介入的机会。API 26 起系统还提供"应用冻屏告警"的统一判定：仅触发 THREAD_BLOCK_3S 或 LIFECYCLE_HALF_TIMEOUT 时，系统判定为冻屏告警（APPFREEZE_WARNING），提供检测、日志抓取和上报能力，帮助在故障发生前定位风险。

判活任务本身优先级很高（VIP 队列），它执行不了只有一种解释：主线程 6 秒钟没有回到事件循环。可能是在跑一个超长的同步任务，可能是死锁，可能是死等某个永远不会来的 IPC 应答。

### APP_INPUT_BLOCK：输入响应超时

检测原理：用户点击应用时，输入系统向应用侧发送点击事件，应用侧的响应反馈回执超时，则上报该故障（出处：《AppFreeze（应用冻屏）检测》）。

超时阈值的版本演变要注意：早期 log 版本 8000ms、nolog 版本 5000ms；从 API version 24 开始，检测阈值统一为 8000ms，不再区分版本。

从 API version 22 开始，日志会输出具体的超时事件信息，格式如：

```text
Wait Event(430) to be marked exceed 8000ms, lastDispatchEvent(430), lastProcessEvent(429), lastMarkedEvent(428)
```

这行信息可以做简单定界：lastDispatchEvent(430) 说明 430 号事件已经分发给应用，lastProcessEvent(429) 说明应用只处理到 429 号——430 号在应用处理环节超时。问题在应用侧。如果几个 id 对不上分发环节，则怀疑系统侧输入分发。

注意一个触发条件上的细节：APP_INPUT_BLOCK 的增强日志（采样栈）以先发生 THREAD_BLOCK_3S 或 LIFECYCLE_HALF_TIMEOUT 为前提。主线程健康的应用，输入超时通常不会被触发——输入事件也是主线程消费的，主线程 8 秒不处理输入，往往早就先触发了 6 秒的 THREAD_BLOCK。所以线上 APP_INPUT_BLOCK 经常与 THREAD_BLOCK 结伴出现。

### LIFECYCLE_TIMEOUT：生命周期切换超时

从 API 26.0.0 开始支持。UIAbility 生命周期切换流程没有在指定时间内执行完毕：AMS（Ability Manager Service）向应用进程发送生命周期切换指令，在固定时间内未收到完成回执，则上报故障（出处：《AppFreeze（应用冻屏）检测》）。

与 THREAD_BLOCK 类似，它也分两级：超过半阈值报 LIFECYCLE_HALF_TIMEOUT 告警，超过完整阈值报 LIFECYCLE_TIMEOUT。不同生命周期的超时时间不同：

| 生命周期 | 超时时间 |
| --- | --- |
| Load | 10s |
| Foreground | 5s |

这类故障指向的代码位置非常具体：onCreate、onWindowStageCreate、onForeground 等生命周期回调里的耗时操作，或者回调里同步等待不返回（比如同步等待网络）。日志的 MSG 字段会带 `ability:EntryAbility background timeout` 之类的说明，并附带 server/client 两侧的生命周期动作时间线，逐条带时间戳，可以直接算出卡在哪一步（9.2 节详述）。

需要额外配置才能拿到这类日志：从 API 26 起，在 AppScope/app.json5 中配置 `DFX_APPFREEZE_LOG_OPTIONS` 环境变量（`report_lifecycle_as_appfreeze:enable`），该类型日志才会以 AppFreeze 类型上报给三方应用，且会计入 AppFreeze 故障指标。

## 根因分类体系

官方把应用冻屏按触发原因划分为 4 类二级根因、17 类三级根因（出处：《应用冻屏故障检测机制及规格说明》）：

| 二级根因 | 三级根因 |
| --- | --- |
| 应用主线程阻塞 | 等锁、同步 Binder 接口调用阻塞、对端 Binder 线程满、同步耗时 I/O 操作、触发长时间 GC、触发长时间抓取内存快照、执行耗时操作 |
| 应用主线程繁忙 | 频繁等锁、频繁调用 Binder 接口、频繁执行 I/O 操作、频繁执行 UI 操作、频繁执行特定业务、执行耗时操作 |
| ArkWeb GPU 进程卡死 | I/O 阻塞、等锁 |
| ArkWeb Render 进程卡死 | 线程执行繁忙、执行耗时 JS |

"阻塞"与"繁忙"的区分值得琢磨：阻塞是主线程停下来等一个东西（锁、Binder 应答、I/O），堆栈上会看到等待点；繁忙是主线程一直在跑但任务太多跑不完，EventHandler 队列里积着大量未处理任务。两者的日志形态不同，9.2 节会对应讲。两类 ArkWeb 根因提醒我们：Web 场景的冻屏可能出在独立的 GPU/Render 进程，分析时先看进程名。

## 与 Android ANR 的对照

从 Android 转过来的读者，可以按这张表建立映射：

| 维度 | Android ANR | HarmonyOS AppFreeze |
| --- | --- | --- |
| 输入超时 | Input dispatching timed out（5s） | APP_INPUT_BLOCK（统一 8s，API 24 起） |
| 主线程 watchdog | 无直接对应（靠各超时项） | THREAD_BLOCK_3S/6S，watchdog 插入判活任务 |
| 生命周期 | Broadcast/Service 超时 | LIFECYCLE_TIMEOUT（Load 10s / Foreground 5s） |
| 日志载体 | /data/anr/traces.txt | /data/log/faultlog/faultlogger/appfreeze-*.log |
| 日志内容 | 全线程堆栈 + CPU 负载 | 堆栈 + EventHandler 队列 dump + BinderCatcher 对端信息 + CPU/内存 + 页面切换轨迹 |
| 事后行为 | 弹"无响应"对话框（用户可等） | 直接强制终止进程 |
| 订阅接口 | 无标准 API | HiAppEvent 订阅 APP_FREEZE 事件 |

两个实质差异要单独强调。其一，Android ANR 弹窗给用户"等待"选项，鸿蒙没有等待选项，判定即杀进程，所以 AppFreeze 对留存的伤害等同于崩溃。其二，鸿蒙日志里的 BinderCatcher 和对端堆栈是 Android traces.txt 没有的东西——主线程卡在同步 IPC 上时，能直接看到对端是谁、等了多久、对端在干什么，这类问题在 Android 上要靠经验猜，在鸿蒙上是读出来的。

## 开发期的检测开关

两个官方提供的开关，调试时必备：

- 通过 DevEco Studio 的 **Debug** 按钮安装并启动应用，系统自动关闭当前工程的超时检测。
- debug 应用可用命令控制：`aa attach -b <bundleName>` 进入调试模式（屏蔽 AppFreeze 检测），`aa detach -b <bundleName>` 退出并恢复检测。需在应用启动后执行，release 应用不支持。

## 常见问题与误区

**"测试机上从来没复现过"。** AppFreeze 强依赖设备状态：低端机、高负载、低内存、热限频都会放大主线程延迟。日志头部从 API 20 起会输出 NOTE 行提示"当前故障可能由系统低内存或热限频引起"，看到它可以先排除应用责任，但反过来也说明——复现要尽量在用户真实机型和负载条件下做。

**"6 秒很长的，我们代码没有 6 秒的操作"。** 单次操作不需要 6 秒。主线程队列里几十个小任务排队、每个 100ms，判活任务排在队尾一样超时。THREAD_BLOCK 指控的是队列 6 秒没轮到判活任务，不是某个任务跑了 6 秒。看 EventHandler dump 里的任务积压情况，区分"一个长任务"和"一堆短任务"。

**"告警事件不用管"。** THREAD_BLOCK_3S 不杀进程，但它是 6 秒故障的前奏。接入 HiAppEvent 订阅 APPFREEZE_WARNING，把告警当成线上灰度指标监控，能在故障率爆发前发现问题。

## 参考资料

- 《AppFreeze（应用冻屏）检测》（appfreeze-guidelines）
- 《应用冻屏故障检测机制及规格说明》（bpta-stability-appfreeze-fault-detection-mechanism）
- 《应用冻屏场景化最佳实践》（bpta-scenario-stability-app-freeze）
- 《应用冻屏事件介绍》（hiappevent-watcher-freeze-events）
- 《应用冻屏告警事件介绍》（hiappevent-watcher-appfreezewarning-events）
- 《aa 工具》（aa-tool）
