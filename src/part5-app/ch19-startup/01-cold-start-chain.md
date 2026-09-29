# 19.1 冷启动链路与阶段拆解

## 三种启动：先分清你测的是哪一种

谈论启动耗时时，冷启动、温启动、热启动经常被混着说，它们的链路完全不同：

- **冷启动**：后台没有该应用的进程，系统需要创建新进程。完整链路从用户点击桌面图标开始，到应用首页首帧渲染完成、数据完全展示为止。这是优化讨论的主战场，也是官方时延规则约束的场景。
- **热启动**：应用进程驻留在内存中，用户再次打开时系统直接从内存恢复状态，无需重新初始化加载资源。耗时最短，通常取决于窗口恢复和页面重绘。
- **温启动**：进程存在，但主实例或页面已被销毁，启动时只需重新创建实例或页面，速度介于两者之间。

测启动耗时前必须控制变量：被测应用先在最近任务里划掉，再进系统设置里"强行停止"，确保进程真的不存在——否则你测的可能是热启动，数据没有任何意义。AppAnalyzer 的"手动性能冷启动体检"流程里专门有一步要求强行停止应用，就是这个原因。

## 冷启动的五个业务阶段

官方《应用冷启动时延优化》最佳实践把冷启动分为五个阶段：

1. **应用进程创建和初始化**：系统完成应用进程的创建和初始化，包括启动页图标（startWindowIcon）的解码。
2. **Application 和 Ability 初始化**：资源加载、方舟虚拟机创建、Application 与 Ability 对象的创建与初始化、依赖模块加载。
3. **AbilityStage/Ability 生命周期**：执行 onCreate、onWindowStageCreate、onForeground 等回调。
4. **加载绘制首页**：加载首页内容，测量布局（Measure/Layout），刷新组件并绘制（Render）。
5. **网络数据二次刷新**：根据业务需求请求网络数据并刷新页面——注意这一阶段在首帧之后，不计入"首帧完成时延"，但计入用户感知的"内容加载完成"。

[图：冷启动五阶段时序，从点击图标到首帧上屏再到数据二刷]

这个分法的作用是把"启动慢"这个模糊抱怨变成五个独立问题：进程创建慢、模块加载慢、生命周期回调慢、首页构建慢、数据到位慢。每个问题的优化手段完全不同。

## Launch 会话的七个量化阶段

DevEco Profiler 的 Launch 场景把上面五个业务阶段进一步切成七个有明确 Trace 边界的阶段，每个阶段对应具体的打点（见《冷启动分析：Launch 分析》与《应用冷启动时延优化》）：

| 阶段 | 含义 | 关键 Trace 点 |
| --- | --- | --- |
| Create Process | 应用进程创建 | `AppMgrServiceInner::StartProcess` |
| Application Launching | 应用启动（attach） | `AppMgrServiceInner::AttachApplication##{bundleName}` |
| UIAbility Launching | UIAbility 启动 | `MainThread::HandleLaunchAbility##{bundleName}` |
| UI Ability OnForeground | 应用进入前台 | `AbilityThread::HandleAbilityTransaction` |
| First Frame - App Phase | 应用侧首帧渲染提交 | `H:ReceiveVsync`、`H:MarshRSTransactionData` |
| First Frame - Render Phase | RS 侧首帧渲染提交 | `H:ReceiveVsync`、`H:RSMainThread::ProcessCommandUni` |
| EntryAbility | 启动完成，首页显示 | —— |

[图：Launch 会话泳道截图，七个阶段的色块排列与 Frame 泳道中的首帧标记]

这套切分的实战价值在于：**每个阶段的所有权不同**。Create Process 是系统的（appspawn fork 与初始化），你只能通过减小 startWindowIcon、减少预装 so 间接影响；Application Launching 到 OnForeground 是模块加载与生命周期回调的地盘，import 治理和延迟初始化在这里生效；First Frame 两个阶段是首页布局复杂度的地盘，用第 20 章的布局优化手段。看到 Launch 泳道后先问"哪段最长"，再翻对应阶段的优化清单，不要在错误的阶段上做无用功。

一个典型分析流程（官方案例）：Launch 录制后发现 UI Ability OnForeground 阶段耗时 3.3 秒，点开该阶段的统计信息，Duration 最长的是 `aboutToAppear`，跳转主线程 Trace 或看 ArkTS Callstack 泳道的函数热点，确认里面有一个 2 亿次的循环计算。把计算挪出生命周期回调后重测——这是冷启动分析的标准动作：阶段 → 函数 → 代码行。

## 官方验收口径：三个数字

性能验收涉及三组口径，别混淆：

- **启动加载完成时延 ≤1100ms**（元服务 ≤340ms）：规则级要求，口径覆盖启动过程动画/视频结束。自测工具是 AppAnalyzer 场景化体检、DevEco Testing 性能测试，云端是 AGC 云测试性能检测。
- **冷启动首帧完成时延**：从点击图标到首帧上屏，官方 Profiler 案例中使用的预期值是 ≤600ms（见《案例：应用首页加载耗时较长导致应用冷启动首帧完成时延不达标》，该案例中超过 800ms 即判不达标）。
- **启动过程动画/视频时延 >3s 建议加进度提示**：这是体验规范中的"建议"级条款，针对启动页播放较长动画/视频的应用。

工程上建议同时盯首帧时延和完成时延：首帧决定"用户觉得打开了没有"，完成时延决定验收过不过。中间这段（首帧到数据二刷完）用骨架屏和缓存数据填充，避免白屏白块。

## 系统侧看一眼：点击图标之后、onCreate 之前发生了什么

冷启动链路的前半段不在你的代码里。Launcher 响应点击后通过 AMS 发起启动请求，AMS 发现进程不存在，通知 appspawn fork 出应用进程——这一段对应 Create Process 阶段，系统侧耗时包括 fork、沙箱与命名空间初始化、启动页图标解码。之后进程加载方舟运行时、执行 HAP 的 abc 字节码入口，进入 Application Launching 阶段。

应用侧能做的有限但不是零：startWindowIcon 的分辨率直接进进程创建阶段的解码耗时（官方建议不超过 256×256，实测 4096×4096 换成 144×144 可省约 37.2ms）；模块数量与 so 数量影响虚拟机初始化和依赖加载。这些数字见 19.2 的清单。

另外系统提供了两个与启动相关的机制值得知道：**应用预加载**（系统根据用户使用习惯，在资源充足时把应用预先加载到特定阶段，开发者配置后由系统决定时机，预加载阶段不能包含界面显示与交互相关操作）和 `aa start -W` 调优命令（API 20+，`aa start` 加 `-W` 可以打印冷启动场景下"系统侧接收到启动请求到 UIAbility 完成首帧绘制"的 TotalTime，是没有 Profiler 时的快速测量手段）。

## 参考资料

- 官方文档：《应用冷启动时延优化》（bpta-application-cold-start-optimization）
- 官方文档：《应用性能体验建议》时延章节（performance-delay）
- 官方文档：《冷启动分析：Launch 分析》（ide-launch-overview）
- 官方文档：《案例：应用首页加载耗时较长导致应用冷启动首帧完成时延不达标》（ide-profiler-launch-case）
- 官方文档：《应用预加载》（preload-application）
- 官方文档：《aa 工具》（aa-tool，-W 参数）
- 官方文档：《启动耗时事件检测》（bpta-performance-startup-time-detection）
