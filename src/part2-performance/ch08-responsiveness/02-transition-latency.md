# 8.2 转场时延分析与优化

## 转场为什么是完成时延的主战场

应用内点击分两种：页面内跳转（Tab 切换、弹窗、局部刷新）和页面间跳转（router/navigation 推新页面）。页面间转场几乎总是完成时延最差的操作——它把"旧页面退场动画、新页面组件树从零创建、新数据网络请求、入场动画"四件事串在一条时间线上，任何一件慢了，900ms 的预算都可能爆掉。

[图：页面转场过程解析图。时间轴上依次排开：点击离手 → 转场动画开始 → 新页面开始创建 → 首帧渲染 → 数据请求与二次刷新 → 页面稳定（完成时延终点）。标出转场动画区间与页面创建区间的交叠关系。]

官方《应用性能体验建议》对转场有两条要求：完成时延达标（应用 ≤900ms），且转场过程流畅（卡顿率为 0ms/s）。本节按"定位流程 → 逐泳道分析 → 优化手段"的顺序展开，所有案例与数据出自官方《点击完成时延分析》。

## 定位流程四步

官方给出的完成时延问题定位流程：

1. **性能体检**：用 AppAnalyzer 的"手动性能页面间转场体检"检测。在应用中找到待检测页面，点击开始，手动执行转场，停止后生成报告。点击完成时延大于 900ms 即判定存在性能问题，报告会列出五类病因（UI 线程方法耗时、组件创建耗时、网络请求耗时、主线程阻塞、图片大纹理，见 8.1 节）。
2. **确定完成时延**：根据体检结果确认耗时是否超规范。
3. **抓取 Trace**：用 DevEco Profiler 的 Frame 模板录制转场过程，按 8.1 节的方法确定起止点（`H:DispatchTouchEvent` type=1 为起点，录屏辅助定终点），在 Profiler 中标记出来。
4. **分析问题**：结合关键泳道 Trace 与 ArkUI Inspector 布局分析定位具体问题。

前三步是机械操作，价值都在第四步。下面按泳道逐条讲。

## 关键 Trace 点：转场场景速查

官方文档给出了一张转场分析的关键 Trace 表，这里按执行顺序重新组织（出处：《点击完成时延分析》）：

| 泳道 | Trace 点 | 含义与关注点 |
| --- | --- | --- |
| 应用线程 | ReceiveVsync | 接收 Vsync 信号，一帧处理的开始 |
| 应用线程 | OnVsyncEvent | 收到 Vsync，渲染流程开始 |
| 应用线程 | FlushVsync | 刷新视图同步事件：记录帧信息、刷新任务、绘制上下文、处理用户输入 |
| 应用线程 | FlushDirtyNodeUpdate | 标脏组件刷新。耗时反映本次刷新波及的组件数量，刷新范围越大越慢 |
| 应用线程 | JSAnimation | 动画执行。动画直接影响组件加载完成时延 |
| 应用线程 | FlushLayoutTask | 布局测算。层级深、组件多就慢 |
| 应用线程 | FlushMessages | 发送消息通知图形侧渲染 |
| 应用进程 | SendCommands | 应用 UI 指令提交到 Render Service |
| 应用线程 | aboutToBeDeleted | 组件析构。未开复用时，FlushDirtyNodeUpdate 和 LazyForEach predict 下会析构组件，刷新时组件重复创建 |
| ArkTS Callstack | createHttp / request | 网络请求的创建与发出。看它离起点有多远，判断请求发起时机 |
| ArkTS Callstack | parse | 数据解析耗时 |
| ArkTS Callstack | off | 取消订阅 |

读法：从起点出发沿时间轴走，每个 Trace 点的耗时就是它那个环节的成本；环节之间的空白（主线程 Runnable/空闲）则提示在等待——等 Vsync、等子线程、等 IPC。

## 四条泳道线，各查一类问题

**ArkTS Callstack：查应用函数耗时。** 这是第一优先级泳道（ArkVM 子泳道优先）。官方案例：分析"HMOS世界"切换 Tab 场景，发现 MainPage 文件中一个匿名函数耗时 350ms，展开后定位到 `AudioPlayerService.getInstance()` 耗时 327ms。看源码发现：Tab 切换的 onChange 回调里，先创建单例、随即又调用 destroy 销毁——一次完全无效的对象创建销毁，挂出 327ms。修复方式是销毁前先判断单例是否实例化过，未实例化直接跳过。这个案例的普遍意义：onChange、onClick 这类高频回调里的每一行代码都直接计入完成时延，值得逐行审视。

**Frame 主线程泳道：查超长帧。** 应用主线程子泳道出现红色帧，说明该帧渲染超时。官方示例：第 145 帧预期耗时 8ms330μs（120Hz），实际耗时 92ms571μs，超出一个数量级，卡顿期间存在很长的 ExecuteJS 调用。转场中一帧吃掉 92ms，完成时延直接失血。首帧红是常见现象，但 90ms 级别的超长帧必须查。

**ArkUI Component：查组件创建成本。** 该泳道记录自定义组件和系统组件的绘制次数与耗时（录制前需手动勾选）。重点关注耗时明显高出同伴的组件，在 Details 区过滤目标组件，看它在创建、布局、渲染各阶段的分布，配合 ArkUI Inspector 看组件树。

**H:Animator：查动画时长。** 转场动画的每一毫秒都计入完成时延。官方做了一组对照实验：Tabs 组件不设 BottomTabBarStyle 时 animationDuration 默认 300ms，分别设为 100ms 和 1000ms，测得完成时延为 99ms39μs 和 1s7ms693μs——动画时长几乎一比一传导到完成时延。需要检查的参数：Tabs 的 animationDuration、Swiper 的 duration、pageTransition 的 PageTransitionOptions.duration。另一个用法：加载中的 loading 动画可以把停止时机与网络请求完成关联，避免数据到了动画还在播。

## 主线程"什么都没干"的时间去哪了

转场 Trace 里最让人困惑的形态：框选区间内主线程长时间空闲（既不在 Running 也没在执行 Trace），完成时延却很长。时间花在哪？官方给了一个 397ms 主线程空闲的诊断案例，方法叫唤醒链回溯：

1. 在主线程空闲区域内，从后往前找 Running 段前面的 Runnable，查看 Runnable 详情中的 WakeUP From Tid——谁唤醒了主线程。
2. 第一次跳转，唤醒方是 VSyncGenerator（Vsync 节拍唤醒，正常）。点击跳到 VSyncGenerator 线程，发现它的 Running 前面是 Sleeping，没有唤醒方——链路断了，回到主线程继续往前找。
3. 第二次找到唤醒方是 OS_FFRT_2_1 线程。跳过去看，该线程连续 Running，说明主线程在等它执行完某个任务。
4. 再回溯发现 OS_FFRT 线程是主线程自己唤醒的——主线程提交了任务到 FFRT 然后等结果。

结论：这 397ms 主要是主线程在等自己提交的子线程任务。优化方向随之明确：要么优化子线程任务的耗时，要么把任务提前执行，让等待与 UI 渲染并行。

这个案例教的不是某个具体 bug，而是一种通用动作：**主线程的空闲段，用 WakeUP From Tid 一段段回溯，总能找到它在等谁**。DevEco Profiler 的 CPU 会话提供 View Integrated Scheduling Chain 开关，开启后点击 CPU 时间片节点可以直接查看完整的唤醒调度链，比手工跳快得多。

## 优化方法清单

按病因对应（出处：《点击完成时延分析》《应用时延优化案例》）：

- **UI 线程方法耗时**：耗时函数移出主线程（TaskPool/Worker）或做缓存；单个函数同步执行建议不超过 15ms（出处：AppAnalyzer 体检规则）。
- **组件创建耗时**：减少嵌套层级；LazyForEach 懒加载；动态 import 按需加载模块；全局自定义组件复用池实现跨页面复用；利用转场动画执行的空闲时间预创建组件。
- **网络请求耗时**：请求提前到点击时刻发出，避免放在异步函数和深层子组件里；预连接、DNS 预解析；数据可缓存时先渲染缓存再二刷。
- **主线程阻塞**：消除"主线程等子线程"的串行依赖，能并行的并行；系统接口避免串行调用；谨慎使用长延时 setTimeout。
- **动画时延**：animationDuration 按业务需要压到合理值，loading 动画与请求完成联动。

## 举一反三

转场分析的四泳道方法可以平移到其他"端到端时延"场景：冷启动（第 14.2 节 Launch 会话，本质是"点击图标"这个转场）、弹窗弹出、下拉刷新、搜索建议出现。套路不变：录屏定起止 → Trace 看分段 → Callstack 找函数 → 空闲段回溯唤醒链。

还有一个容易被忽略的维度：转场不仅要看"多快"，还要看"卡不卡"。900ms 达标但中间连续丢帧的转场，体感依然差。Frame 泳道的 Lost Frames 和 Hitch Time 子泳道可以量化这一段流畅度，对应体验建议里"转场卡顿率 0ms/s"的指标。

## 参考资料

- 《点击完成时延分析》（bpta-click-to-complete-delay-analysis）
- 《点击响应时延分析》（bpta-click-to-click-response-optimization）
- 《应用时延优化案例》（bpta-application-latency-optimization-cases）
- 《Frame 分析》（ide-insight-session-frame）
- 《CPU 活动分析》（ide-insight-session-cpu）
- 《应用性能体验建议》（performance-delay / performance-frame-rate）
