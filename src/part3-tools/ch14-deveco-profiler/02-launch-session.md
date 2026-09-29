# 14.2 Launch 会话：启动性能分析

## 这个工具解决什么问题

冷启动慢是用户流失的头号原因之一，但"启动慢"三个字太粗——慢在进程创建、Application 初始化、Ability 生命周期、首页面加载还是首帧渲染，解法完全不同。DevEco Profiler 的 Launch 会话把应用冷启动过程拆成阶段，抓取各阶段耗时数据，帮助快速定位瓶颈阶段（出处：《Launch 分析》《应用冷启动优化》）。

录制入口与 Frame 一致：Profiler 面板选设备与应用 → 选 Launch 模板创建 Session。几个录制前提要注意（出处：《Launch 分析》《Profiler 错误码》）：

- 启动模式分**自动启动**（Profiler 自动拉起应用）和**手动启动**（自己在桌面点图标），可点击图标切换。
- 锁屏状态下也可以录制。
- 命令行（aa start）拉起的 Release 应用不能做 Launch 分析。
- 手动启动录制过程中设备断开连接会报"App not started correctly"，重连后重新录。

## 启动阶段拆解

Launch 模板把冷启动过程分为以下阶段（出处：《Launch 分析》《性能 FAQ-11》）：

| 阶段 | 说明 |
| --- | --- |
| Process Creating | 应用进程创建 |
| Application Launching | 应用启动（Application 初始化） |
| Ability Launching | Ability 创建与生命周期 |
| Page Loading | 首页面加载与首帧渲染 |

[图：Launch 会话时间线。横向按阶段着色的条带：Process Creating → Application Launching → Ability Launching → Page Loading，终点对齐首帧上屏。条带下方是各阶段对应的 Trace 泳道。]

阶段的边界让我们能先把"启动慢"收敛到一个区间，再进区间内部看 Trace——这和丢帧分析"先定界再深入"是同一套思想。

## 核心指标：冷启动首帧完成时延

定义：从用户点击桌面应用图标离手开始，到应用进程首帧绘制结束的时间（出处：《Launch 分析案例》）。

**找起点**：在 Trace 中搜索 `H:DispatchTouchEvent`，找到 type=1（离手）的那条。要注意辨别这个事件落在桌面（sceneboard）进程还是应用进程——点击图标那一刻应用进程还没创建，事件在桌面侧。

**找终点**：应用进程启动后收到首个垂直同步信号时，会通知 Render Service 进行图形渲染。标志是三个首尾相连的短 Trace：`H:FlushMessages` > `H:SendCommands` > `H:MarshRSTransactionData`。这三个点耗时极短，肉眼在时间线上找不到，官方给的操作是：框选应用主线程的 `H:ReceiveVsync` 区段，点击搜索框选 Search Units Data，输入 FlushMessages 回车定位（出处：《Launch 分析案例》）。

**量耗时**：框选起止两点，读出区间长度即为首帧完成时延。

**定标准**：官方案例中预期冷启动首帧完成时延不超过 600ms，实测框选后超过 800ms，判定不达标（出处：《Launch 分析案例》）。

## 案例分析：800ms 慢在哪

官方《Launch 分析案例》的排查过程是 Launch 会话的标准用法：

1. 框选起止点，确认 800+ms 超预期的 600ms。
2. 切到应用进程的 Process 泳道，看主线程（线程号与进程号一致）的 Trace。
3. 在阶段时间线内找耗时异常的任务，结合 ArkTS Callstack 逐层展开——本例定位到 Ability 生命周期回调函数中执行了耗时操作。

这个案例揭示的规律：冷启动超标的第一嫌疑区是 Ability/AbilityStage 生命周期回调（onCreate、onWindowStageCreate）里的同步耗时——初始化 SDK、读文件、同步请求。第二嫌疑区是首页面一次性渲染过多内容（没有懒加载、没有分包）。验证方法都是同一个：把可疑段的耗时从总时延里减掉，看账能不能对上。

**热启动的区别**：热启动时进程已存在，Process Creating 和大部分 Application 初始化被跳过，耗时集中在 Ability 前台切换与页面恢复。官方对热启动的度量是"到 UIAbility 状态切换至前台"，标准比冷启动宽松，但如果热启动接近冷启动耗时，常见原因是 onForeground/onWindowStageRestore 里做了全量重建，或者内存压力导致进程实际已被回收、名义上的热启动实际是冷启动——后者在 Process 泳道上看进程创建时间即可分辨。

Frame 泳道在 Launch 模板里同样可用：点选 Frame 泳道，Details 区展示启动动效详情，More 区展示动效帧的 Animation Data List（出处：《Frame 分析》）。启动动画本身的耗时也计入首帧时延，别忘了看。

## 辅助手段：aa 命令粗测

不需要完整 Trace、只想快速拿个数字时，`aa start` 的 `-W` 参数（API 20 起）可以测量启动耗时：

```shell
aa start -b 包名 -a Ability名 -W
```

返回中的 `TotalTime`：冷启动场景下是系统侧收到启动请求到 UIAbility 完成首帧绘制的耗时（ms）；热启动场景下是到前台的耗时。`StartMode` 字段标明本次是 Cold 还是 Hot（出处：《aa 工具》）。它适合写进脚本做回归——每次构建跑十遍取中位数，发现劣化再上 Profiler 细查。注意 -W 只对显式启动（带 -b 和 -a）生效。

## 常见问题与坑

**终点找错**。`H:MarshRSTransactionData` 在应用每次提交渲染时都会出现，启动过程可能有多次，要取首个 Vsync 后的第一组三连点。用搜索定位比肉眼可靠。

**把进程创建时间算进应用账上**。Process Creating 阶段是系统行为，应用能优化的区间从 Application Launching 开始。汇报启动指标时建议分开给：总时延 + 应用侧可控时延。

**热启动当冷启动测**。测试前确保应用进程不存在（`aa force-stop 包名` 或清理后台），否则测的是热启动，数据没有意义且会偏乐观。

**录到了但没数据**。手动模式下点击图标太晚（录制已开始很久才点）或太早（录制未就绪），都会导致关键段落缺失。听工具提示操作：等"开始录制"就绪再点图标。

## 参考资料

- 《Launch 分析》（ide-insight-session-launch）
- 《Launch 分析案例》（ide-profiler-launch-case）
- 《应用冷启动优化》（bpta-application-cold-start-optimization）
- 《aa 工具》（aa-tool）
- 《性能 FAQ-11》（faqs-performance-11）
- 《Profiler 错误码》（ide-profiler-errorcode）
