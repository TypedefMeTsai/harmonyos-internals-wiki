# 附录 C：HiTrace tag 与关键 Trace 点速查

## tag 速查表

`hitrace -l` 列出当前设备全部可用 tag，以下是性能分析常用集合（不同版本可能有增减，以设备实际输出为准）：

| tag | 覆盖内容 | 典型使用场景 |
| --- | --- | --- |
| app | 应用侧 HiTraceMeter 打点（startTrace/finishTrace） | 自定义关键段耗时 |
| ace | ArkUI 框架（组件构建、布局、状态更新） | 丢帧、布局耗时、BuildLazyItem |
| ability | Ability 框架（生命周期、进程管理） | 启动分析、生命周期耗时 |
| graphic | 图形栈（渲染、合成相关） | 渲染管线分析 |
| render_service | Render Service 侧任务 | RenderDeadlineMissed、合成耗时 |
| window | 窗口管理 | 窗口创建、焦点切换 |
| binder | binder IPC | 跨进程调用耗时、AppFreeze 锁等待排查 |
| sched | 调度事件 | 线程调度延迟、优先级问题 |
| freq | CPU 频率 | 降频判断、发热场景 |
| disk | 块设备 I/O | 存储耗时 |
| ohos | 系统服务通用 | 系统侧综合 |
| appspawn / zygote 类 | 进程孵化 | 冷启动 Create Process 阶段 |
| irq / workqueue / mmc 等内核 tag | 内核态事件 | 深度系统分析 |

日常抓应用性能问题的基础组合：`hitrace -t 10 -o out.htrace app ace ability graphic render_service window binder sched freq`

## 关键 Trace 点速查表

名称格式说明：`H:` 前缀表示 HiTraceMeter 打点；其余为系统框架内打点。

### 冷启动链路（第 19 章）

| Trace 点 | 含义 | 异常指向 |
| --- | --- | --- |
| AppMgrServiceInner::StartProcess | 应用进程创建开始 | 过长 → 进程创建慢（图标解码、预载 so 多） |
| AppMgrServiceInner::AttachApplication##{bundleName} | 应用 attach 阶段 | 过长 → 虚拟机/资源初始化慢 |
| MainThread::HandleLaunchAbility##{bundleName} | UIAbility 启动 | 过长 → 模块加载、onCreate 耗时 |
| AbilityThread::HandleAbilityTransaction | Ability 事务处理（进前台） | 过长 → 生命周期回调阻塞 |
| H:SourceTextModule::Evaluate | ets 模块求值 | 频繁/耗时 → import 治理（19.2） |
| H:JSPandaFileExecutor::ExecuteFromAbcFile | abc 字节码加载执行 | 大模块、嵌套 export * |
| H:ReceiveVsync | 收到 Vsync | 首帧定位锚点 |
| H:MarshRSTransactionData | 应用侧提交渲染事务 | 首帧 App 阶段标志 |
| H:RSMainThread::ProcessCommandUni | RS 处理渲染指令 | 首帧 Render 阶段标志 |

### 帧渲染与丢帧（第 7、20 章）

| Trace 点 | 含义 | 异常指向 |
| --- | --- | --- |
| H:OnVsyncEvent / ReceiveVsync | Vsync 到达 | 帧周期锚点 |
| H:FlushBuild / FlushLayoutTask / FlushRenderTask | 应用侧构建/布局/渲染任务 | 耗时 → 节点多、状态更新范围大（20.1） |
| H:BuildLazyItem | 懒加载列表项创建 | 滑动中出现长耗时 → 未开组件复用、item 布局复杂 |
| H:BuildRecycle | 复用组件更新 | 正常应极短（官方实测约 0.2ms 级） |
| AppDeadlineMissed | 应用侧未按期完成帧 | 红帧，应用侧优化方向 |
| RenderDeadlineMissed | RS 侧未按期完成 | RS/GPU 侧优化方向（视效、合成） |
| H:RosenWeb | Web 侧待提交 buffer 数据量 | 某周期为 0 → Web 侧丢帧（20.3） |
| JSView:ExecuteRerender 类 | Web 前端 JS 重渲染执行 | 长耗时 → JS 阻塞致 RosenWeb 无产出 |

### 无响应与阻塞（第 9 章）

| Trace 点 | 含义 | 异常指向 |
| --- | --- | --- |
| THREAD_BLOCK（AppFreeze 事件） | 主线程阻塞型冻屏 | 主线程同步 I/O、锁、binder |
| binder 事务相关打点 | 同步 binder 调用 | 等待对端进程响应超时 |

### 并发与任务（第 5 章）

| Trace 点 | 含义 | 异常指向 |
| --- | --- | --- |
| TaskPool 任务执行打点 | 任务池任务调度执行 | execute 到执行间隔长 → 池满/优先级低 |
| Worker 消息打点 | Worker 线程消息处理 | 长任务阻塞 Worker |

## 使用备忘

- tag 与 Trace 点名称随版本演进，以当前设备 `hitrace -l` 与 Profiler 泳道为准。
- hitrace 级别阈值（`--trace_level`）控制打点输出，商用版本部分调试级打点不输出。
- 自定义打点命名建议带业务前缀（如 H:MyApp:LoadFeed），便于在泳道中检索。
- 单条 trace 总长度 512 字节上限，name+customCategory+customArgs 建议 ≤420 字节。
