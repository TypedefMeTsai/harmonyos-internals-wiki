# 15.2 FaultLog、CppCrash 与应用事件

## FaultLog：故障档案库

应用崩溃（JsCrash/CppCrash）或冻屏（AppFreeze）后，系统的 FaultLogger 模块收集故障信息、生成结构化日志、存到设备 `/data/log/faultlog/faultlogger/` 目录。文件名按类型区分（出处：《jscrash 日志获取》《AppFreeze 日志获取》）：

```text
jscrash-进程名-进程UID-毫秒级时间.log
cppcrash-进程名-进程UID-毫秒级时间.log
appfreeze-进程名-进程UID-毫秒级时间.log
```

三种获取途径，与第 9.2 章所述一致：

1. **DevEco Studio FaultLog 窗口**：自动收集设备日志，按"包名 > 故障类型 > 故障时间"组织，关键信息高亮；debug 包可直接从堆栈跳转代码行。DevEco Studio 6.0.0 Beta1 起，CppCrash 类型日志支持结构化展示（Fault Analysis 页签）和日志过滤，下方分 Stacks（线程堆栈）和 Logs（HiLog 日志）页签；AppFreeze 日志额外有 Binder Communication、System 页签。
2. **hdc 命令**（需开发者选项）：

```shell
hdc file recv /data/log/faultlog/faultlogger 本地目录
```

3. **代码内订阅**：HiAppEvent 订阅 APP_CRASH / APP_FREEZE 事件，从 `external_log` 字段拿日志文件路径（下文详述）。

API 22 起还可以用 `hidumper -e --print [进程名]` 直接打印异常退出故障日志，支持 `--since/--until` 时间窗（出处：《hidumper》）。

## CppCrash 日志解读

Native 崩溃（C/C++ 层）是故障分析里信息密度最高的一类。进程收到致命信号后，系统抓取崩溃现场：进程基本信息、故障信号、寄存器快照、调用栈回溯、内存 maps（出处：《CppCrash 故障检测机制》）。逐段看：

**Reason（信号）**：日志的定性入口。常见的有——

| 信号 | 含义 | 典型原因 |
| --- | --- | --- |
| SIGSEGV | 段错误 | 空指针/野指针解引用、栈溢出 |
| SIGABRT | 异常终止 | assert 失败、检测到堆损坏后主动 abort |
| SIGBUS | 总线错误 | 未对齐的内存访问 |
| SIGFPE | 算术异常 | 除零 |
| SIGILL | 非法指令 | 跳转到数据区执行、指令集不匹配 |
| SIGSTKFLT | 栈错误 | 协处理器栈错误，现代系统基本废弃 |
| SIGSYS | 非法系统调用 | Seccomp 沙箱拦截 |

**寄存器与 Memory near registers**：崩溃时的寄存器快照，以及各寄存器指向地址附近的内存内容。官方场景化案例演示了用法：memcpy 崩溃时，从寄存器中找到三个参数寄存器的值，结合崩溃地址 0x0000005b3b46c000，确认出问题的是 memcpy 第 2 个参数（源地址），再通过 Memory near registers 查看该地址附近的内存特征，判断是未映射还是已释放（出处：《CppCrash 场景化分析》）。

**调用栈**：崩溃线程的堆栈回溯，格式与第 9 章 AppFreeze 堆栈一致（`#00 pc 地址 库路径(符号+偏移)`）。debug 包在 DevEco Studio 里直接跳转源码；release 包混淆后需要 **hstack** 工具配 Source Map 把堆栈还原为源码调用栈，支持 Windows/Mac/Linux（出处：《CppCrash 分析方法》）。

**SubmitterStacktrace**：异步线程崩溃时，系统会把提交该异步任务的线程栈也打出来，与崩溃线程栈以此字符串分隔。用于定位"任务本身没问题、是提交者传了坏数据"的场景，默认仅在 ARM 64 位系统开启（出处：《jscrash 日志规格》）。

分析流程建议（出处：《CppCrash 分析方法》）：看 Reason 定性 → 看栈顶定位崩溃函数 → hstack 还原行号 → 结合寄存器与附近内存判断非法访问类型 → 回到业务代码检视上下文 → 借助 hilog 崩溃现场日志还原业务场景。还有一个 C 侧的经典坑：信号处理函数里调用了非异步信号安全函数（如 malloc），会导致二次崩溃或抓栈失败——信号处理函数里只允许用栈上对象和最简操作（出处：《CppCrash 优化实践》）。

## hiAppEvent：故障订阅与事件打点

HiAppEvent 是系统提供的事件订阅与打点机制，支持故障、统计、安全、行为四类事件（出处：《HiAppEvent 介绍》）。两个用法：

**订阅系统故障事件**：注册 Watcher 监听崩溃、冻屏：

```typescript
hiAppEvent.addWatcher({
  name: "AppCrashWatcher",
  appEventFilters: [{
    domain: hiAppEvent.domain.OS,
    names: [hiAppEvent.event.APP_CRASH, hiAppEvent.event.APP_FREEZE]
  }],
  onReceive: (domain, appEventGroups) => {
    for (const group of appEventGroups) {
      for (const info of group.appEventInfos) {
        const logs = info.params['external_log'] as string[];
        // 读取故障日志文件，上报或本地分析
      }
    }
  }
});
```

APP_FREEZE 事件的参数除 external_log 外，还带 exception（故障名与 message）、hilog（故障进程最近最多 100 行日志）、event_handler（主线程未处理消息）、threads、memory 等结构化字段（出处：《应用冻屏事件介绍》）——很多分析不需要打开日志文件就能完成。崩溃事件的采集规格可通过 `hiAppEvent.configEventPolicy` 调整：页面切换日志开关、扩展打印 pc/lr 寄存器附近内存、日志截断大小、精简 maps、minidump 采集（出处：《订阅崩溃事件》）。

**自定义打点**：`hiAppEvent.write(事件名, 事件类型, 参数)` 记录业务事件，`setEventParam` 追加公共参数。适合把关键业务路径（下单、支付）的轨迹打进事件流，故障时随 external_log 一起还原现场。

## errorManager：未捕获异常的最后一道防线

`errorManager.on('error', observer)` 注册错误观测器，捕获应用产生的 JS_CRASH（未捕获异常）。观测器捕获到异常时应用不退出——官方建议在回调里完成日志上报后增加同步退出操作（出处：《errorManager》）。与 appRecovery 配合可以实现"保存现场并重启应用"：

```typescript
let callback: errorManager.ErrorObserver = {
  onUnhandledException(errMsg) {
    console.error(errMsg);
    appRecovery.saveAppState();
    appRecovery.restartApp();
  }
};
registerId = errorManager.on('error', callback);
```

注意它只在主线程注册使用；Worker 线程出错会抛错误码，用 try-catch 处理。只要错误日志不需要拦截退出的话，优先用 HiAppEvent 订阅，errorManager 留给"必须挽救一下"的场景（出处：《errorManager 开发指导》）。

## 聚类：把一千份日志变成几个问题

线上回收的故障日志动辄成百上千份，逐份看不可能。官方建议按**故障线程调用栈**聚类：相同栈即相同根因，版本号、时间戳、设备信息全部忽略。CppCrash 与 AppFreeze 聚类方法一致，故障栈中含 IPC 栈帧时补充 binder 堆栈做特征（出处：《AppFreeze 聚类》《CppCrash 聚类》）。实际做聚类时提取栈顶连续 N 个有效帧做哈希，就能把日志收敛成按根因排序的问题清单——先修占比最高的那个。

## 常见问题与坑

**设备上 /data/log/faultlog 是空的**：开发者选项没开，或故障根本没发生（应用是被系统回收而不是崩溃——查 hilog 里的 LowMemoryKill 记录）。

**release 包堆栈全是地址没有符号**：忘了保留对应版本的 Source Map 或符号表。每次发布构建必须归档混淆映射文件，hstack 离了它什么都还原不了。

**FaultLog 里有日志但 DevEco Studio 不显示**：IDE 收集有延迟，手动刷新或直接用 hdc file recv 拉。

**errorManager 回调里做了异步上报，日志丢了**：回调返回后进程可能立刻退出，异步操作来不及完成。上报要同步做完，或写入本地文件待下次启动补传。

## 参考资料

- 《CppCrash 检测与日志规格》（cppcrash-guidelines）
- 《CppCrash 故障检测机制》（bpta-stability-cppcrash-fault-detection-mechanism）
- 《CppCrash 分析方法》（bpta-stability-app-crash-cpp-way）
- 《CppCrash 场景化分析》（bpta-scenario-stability-cppcrash）
- 《FaultLog 使用》（ide-fault-log / ide-faultlog-cppcrash / ide-faultlog-appfreeze）
- 《jscrash 日志规格》（jscrash-guidelines）
- 《HiAppEvent 介绍》（hiappevent-intro）
- 《订阅崩溃事件》（hiappevent-watcher-crash-events）
- 《errorManager》（js-apis-app-ability-errormanager / errormanager-guidelines）
- 《堆栈解析工具 hstack》（ide-command-line-hstack）
- 《hidumper》（hidumper）
