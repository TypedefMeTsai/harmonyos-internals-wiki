# 11.2 长时任务与唤醒治理

## 后台任务的四种正规通道

应用退至后台后想继续运行，唯一合法的方式是 Background Tasks Kit 提供的四类受约束任务（出处：《后台任务概述》《后台任务术语》）：

**短时任务（TransientTask）**：实时性要求高、耗时不长的任务，如保存用户正在编辑的笔记、发送一条消息。机制是配额制（Transparency Quota）：每个应用有短时任务时间配额，单次申请有时长上限，配额耗尽后不允许再申请；可用 `getRemainingDelayTime` 查询本次申请的剩余时间。规则要求：单次配额超时又不取消任务，应用进程会被终止；任务完成后要及时取消。所以短时任务的正确写法是"申请 → 干活 → 立刻 cancel"，把配额当稀缺资源。

**长时任务（ContinuousTask）**：长时间运行在后台、用户可感知的任务——后台播放音乐、导航、录音、设备连接、运动记录等。三个合规要点：仅允许按声明的原定用途使用；任务可由用户主动开始和停止；执行期间用户可感知（通常体现为通知栏状态）。申请部分模式（如特殊场景媒体处理）需要向应用市场提交说明，官方给出了申请话术模板（出处：《长时任务申请指导》）。用途外滥用长时任务是上架审核的重点打击对象。

**延迟任务（WorkScheduler）**：时效性要求不高的任务——有网络时不定期拉邮件、闲时同步数据。应用设定触发条件（网络类型、充电状态、存储状态、电池状态、定时状态等），系统根据内存、功耗、设备温度、用户使用习惯统一调度时机，到点拉起应用执行。它是对应用最友好的唤醒方式：系统会把多个应用的延迟任务凑在一起批量执行，减少唤醒次数。能用延迟任务的场景，就不要用轮询。

**代理提醒（ReminderAgent）**：日历提醒、闹钟这类到点必达的场景，由系统代理弹出提醒，应用不需要在后台守着。

选型逻辑一句话：用户感知得到的连续活动用长时任务，几秒能完的事用短时任务，不着急的用延迟任务，到点提醒用代理提醒。四种都不满足的"后台需求"，大概率本身就不合规。

[图：后台任务选型决策树。任务是否需要持续运行？→ 长时任务；几秒内完成？→ 短时任务；可延迟可批量？→ WorkScheduler；定时提醒？→ 代理提醒。]

## 唤醒：功耗的隐形大头

比"后台跑什么"更隐蔽的问题是"设备被叫醒多少次"。每次唤醒——CPU 从 idle 拉起、射频从休眠恢复、屏幕点亮——都有一次性成本，高频次的小唤醒累积起来比一次长任务更费电。应用的轮询定时器、高频心跳、无序的后台同步，都是唤醒源。治理方向：

- 用 WorkScheduler 替代自有轮询，让系统合并唤醒；
- 网络心跳降频、合并，服从系统的 doze 节奏；
- 退后台前注销不再需要的监听和回调。

还有一类唤醒藏得更深：Vsync 请求。它不发通知、不占网络，但能让渲染流水线在静止页面上持续空转。这是本节下半部分的主题。

## Vsync 的发车模型

HarmonyOS 中不同来源的绘制请求统一由 Render Service 管理，Render Service 依赖 Vsync 实现全局统一的信号收发。官方《Vsync 低功耗优化》用了一个公交系统的类比：VsyncGenerator 子进程是始发站，RSHardware 子进程（屏幕显示）是终点站，每个需要 Vsync 的申请者买一张"车票"，VsyncGenerator 统计申请者信息，确定发车频率（30/60/120Hz），经绘制合成后反映在屏幕刷新率上。

这个模型的功耗含义：只要车上还有一个乘客，车就得按点发。一个静止页面上被遗忘的 Vsync 申请者，会让整条渲染管线——应用主线程、RS、GPU——以 60Hz 持续空转。

在 Profiler 抓 Trace 后，用三个关键字可以把 Vsync 的申请和消费关系看清楚（出处：《Vsync 低功耗优化》）：

- **`H:SendVsyncTo conn: xxx`**：在 VsyncGenerator 进程中，显示本次发车的乘客名单。一段时间内反复搜索它，就知道有哪些绘制请求者、各自什么频率。
- **`H:xxx_RequestNextVsync`**：申请者 xxx 请求了下一帧车票，通常出现在 RS 的 OS_IPC 线程。静止场景下谁还在持续请求，谁就是嫌疑对象。
- **`H:ReceiveVsync name:xxx`**：申请者消费了一帧 Vsync。ArkUI 组件刷新对应 `ReceiveVsync name:WM_[进程号]`；React Native、Flutter 等自渲染框架依赖进程下的 OS_VsyncThread 接收。

分析动作：把 VsyncGenerator 与被拉起的进程置顶排列，按 ReceiveVsync 的 name 匹配绘制对象。不同来源的申请者拿到的 Vsync 周期可以不同，各自按需取用。

## 案例：displaySync 空刷

官方案例（出处：《Vsync 低功耗优化》）。某页面用 displaySync 实现 Canvas 弹幕，弹幕显示时以 60Hz 刷新，用户关闭弹幕后，Trace 上依然能看到 displaySync 对象持续执行业务。问题帧的三个特征：

- `H:DispatchDisplaySync` 下挂着 DisplaySyncId[327]，持续 4ms，附带 `DrawFPS[60] VsyncRate[60]`；
- 该 UI 帧"左大右小"，`FlushRenderTask` 下没有任何 Trace——本帧没有任何 ArkUI 脏区任务；
- `FlushMessage` 下没有 `H:MarshRSTransactionData` 送出——绘制内容在 UI 侧就被拦截，根本没提交给 RS，纯空跑。

展开 DisplaySync 内部还能看到大量 CanvasRenderingContext2D 业务调用；`H:ViewPU.viewPropertyHasChanged` 会打印涉及的组件名，帮助定位到具体组件（这个打点需要先开 debug 开关：`hdc shell param set persist.ace.debug.enabled 1` 并重启）。

**修复**：displaySync 对象与组件生命周期绑定，两处兜底——

```typescript
// 组件销毁时停止并置空，防止空跑与泄漏
aboutToDisappear() {
  if (this.backDisplaySyncSlow) {
    this.backDisplaySyncSlow.stop();
    this.backDisplaySyncSlow = undefined;
  }
}

// 组件不可见时暂停
.onVisibleAreaChange([0.0, 1.0], (isExpanding: boolean, currentRatio: number) => {
  if (!isExpanding && currentRatio <= 0.0) {
    this.backDisplaySyncSlow?.stop();
  }
})
```

**效果**：修改后的 Trace 中，用户点击 tab 切换页面、解注册 displaySync 对象后，UI 帧率降至 0Hz，空刷消失。

另一个同类案例来自 NativeVsync：某 React Native 应用的静置页面上，OS_VsyncThread 线程持续收到 `H:ReceiveVsync name:TaroAnimation / dirty`，框架默认的 onUITick 帧回调以 60Hz 请求 Vsync（该问题在 react-native-harmony 0.72.70 后续版本修复）。这个案例的教训：RN、Flutter、Taro 这类框架和应用自定义的 `postFrameCallback` 回调都可能成为隐藏乘客，分析时别只盯 ArkUI 组件。

## 降帧与让路：主动降低刷新成本

除了消灭空刷，还可以主动告诉系统"我不需要那么高帧率"：

- displaySync / NativeVsync 创建时传入 ExpectedFrameRateRange，让内容按 30Hz 等低帧率运行（视频、弹幕、仪表盘类内容足够）。注意不要把 expected/min/max 都设成 120——那会干扰系统可变帧率机制，制造不必要的负载（出处：《高负载组件渲染优化》）。
- XComponent 自绘制内容可独立申请绘制帧率（出处：《低功耗体验设计》）。
- 页面内容长时间无变化时停止帧回调，有滑动或动画时再开启——这是官方对一切帧回调类接口的总建议。

## 常见问题与误区

**"静止页面 Vsync 空刷不影响用户，不用管"。** 空刷让 CPU/GPU 持续低频工作，直接吃掉续航；上架检测和功耗体检都会抓。它属于"用户感知不到、但代价真实存在"的问题，恰恰是功耗优化的主战场。

**"申请个长时任务保活最省事"。** 长时任务有用途白名单和用户感知要求，滥用会被审核拒绝；能延迟的活交给 WorkScheduler，系统批量调度比应用自己保活省电得多。

**"帧率拉满体验最好"。** 内容与帧率匹配才最好。30fps 的视频内容在 120Hz 下播放，多出来的刷新都是浪费；ExpectedFrameRateRange 存在的意义就是让帧率回到内容需要的水平。

## 参考资料

- 《Vsync 低功耗优化》（bpta-vsync-power-optimization）
- 《后台任务概述》（background-task-overview）
- 《后台任务术语》（background-tasks-glossary）
- 《延迟任务 WorkScheduler》（work-scheduler）
- 《后台任务合理使用》（standard-background-task）
- 《长时任务申请指导》（bgtask-design-formula）
- 《高负载组件渲染优化》（bpta-dispose-highly-loaded-component-render）
- 《低功耗体验设计》（bpta-power-consumption-experience）
