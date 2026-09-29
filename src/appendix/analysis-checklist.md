# 附录 D：性能分析 Checklist

按问题类型组织。每类的思路一致：先量化、再定位、后验证。工具简称：Frame＝DevEco Profiler Frame 会话，Launch＝Launch 会话，Snapshot＝内存快照。

## 流畅性（丢帧/卡顿）

1. 复现并用 Frame 会话录制问题场景（滑动、转场、动画）。
2. Frame 泳道找红帧，区分类型：AppDeadlineMissed（应用侧）还是 RenderDeadlineMissed（RS 侧）。
3. 应用侧：展开该帧的 FlushBuild/FlushLayoutTask/FlushRenderTask，找耗时段：
   - BuildLazyItem 长 → 列表项复杂/未开复用（20.1）；
   - FlushLayoutTask 任务数多 → 节点多/缺布局边界（20.1）；
   - 状态更新触发的重建范围大 → 状态拆分（20.1）。
4. RS 侧：看 RenderFrame 与合成方式：
   - 实时模糊/高阶视效 → 换静态 blur、合并视效（20.2）；
   - 视频/自渲染图层 + 视效交叠 → 排查 HWC 失效（20.2/20.3）。
5. Web 页面丢帧：ArkWeb 泳道看 RosenWeb 是否为 0、JS 线程长任务（20.3）。
6. 修复后同场景复测：丢帧率、单帧耗时对比，纳入回归。

## 响应速度（点击慢/转场慢）

1. 口径对齐：点击响应时延 ≤100ms、点击完成时延 ≤900ms（应用内，体验规范）。
2. 确定慢在哪段：响应（给出第一反馈）还是完成（内容就绪）。
3. 响应慢：查点击到第一帧之间主线程在做什么（同步 I/O？重计算？锁？）。
4. 完成慢：拆解为 页面构建 + 数据等待 + 渲染：
   - 页面构建重 → 布局优化（20.1）；
   - 数据等待 → 请求提前、缓存预填、预连接（19.2/22.2）；
   - 转场动画卡顿 → 用系统转场，检查 expectedFrameRateRange（20.2）。
5. 验证：HiTraceMeter 打点或 AppAnalyzer 点击时延体检复测。

## 无响应（AppFreeze）

1. 取故障数据：FaultLog（/data/log/faultlog）或 hiAppEvent 的 APP_FREEZE 事件。
2. 看异常类型：THREAD_BLOCK（主线程阻塞）、其它类型按第 9 章分类。
3. THREAD_BLOCK 排查三问：
   - 主线程有没有同步文件/数据库 I/O（22.1）？
   - 有没有主线程等待锁/等待 binder 对端（看 peer_binder 字段）？
   - 有没有超长同步计算（看 threads 全量栈）？
4. 修复后压测复现场景；线上关注 appFreezeWarning 先行指标（24.1）。

## 内存

1. Memory 泳道看曲线形态：台阶上升（泄漏）/ 高位平稳（膨胀）/ 尖峰（瞬时分配）。
2. 泄漏：Snapshot 对比工作流——基线快照 → 操作 N 次 → 强制 GC → 再快照 → 个数增量排序 → Retainers 追引用链（21.1）。
3. 重点嫌疑：订阅未 off、单例持 context、事件总线、无上限缓存、napi_create_reference（21.1）。
4. 膨胀：模块内存预算审计；图片 PixelMap 治理（解码尺寸、及时释放、缓存上限）。
5. 尖峰：Allocation 分析抓分配栈，对号大图解码/全量加载/大 JSON。
6. 验证：反复操作后 PSS 回基线；Snapshot 对比增量稳定；长时运行无趋势上涨。

## 功耗与发热

1. AppAnalyzer 功耗/后台规则体检，先消违规项（23.1）。
2. 后台：CPU 负载是否超线（短时任务 8%、长时任务 10 分钟 80%）；退后台后传感器/蓝牙/麦克风/持锁是否释放。
3. 前台：
   - 静止页面是否仍有帧任务（Frame 会话 Vsync 泳道）；
   - 动画帧率是否可降（expectedFrameRateRange / displaySync）；
   - 视频是否硬解、弹幕是否硬加速；
   - 自渲染图层与模糊是否交叠（HWC 失效）。
4. 发热伴随卡顿：按"帧耗时 → 帧率 → 合成路径"顺序治理，再谈温控（23.1）。
5. 验证：功耗工具对比电量曲线；GPU/CPU 负载与温升回归。

## 启动

1. 口径对齐：启动加载完成时延 ≤1100ms（元服务 ≤340ms）；首帧时延参考 ≤600ms。
2. 测前控制变量：强行停止应用，确保真冷启动。
3. Launch 会话切七阶段，找最长段：
   - Create Process → startWindowIcon 规格；
   - Application Launching → import 治理、HAR/HSP 选型；
   - UIAbility/OnForeground → 生命周期回调耗时；
   - First Frame → 首页布局；
   - 之后 → 网络请求时机、缓存预填（19.2 清单逐条对）。
4. 验证：Launch 复测 + AppAnalyzer 冷启动体检报告全绿。
