# 附录 E：术语表

按拼音/字母顺序排列。每条一句话定义，需要展开处标注对应章节。

## A

- **AbilityStage**：HAP 模块级的组件管理器，每个 HAP 首次加载时创建，onCreate 适合放模块级初始化（1.3、19 章）。
- **AGC（AppGallery Connect）**：华为应用全生命周期服务平台，含 APMS 质量管理、云测试性能检测（24.1）。
- **APMS**：AGC 的应用质量管理服务，提供崩溃聚类、启动和卡顿分析等线上报表（24.1）。
- **AppAnalyzer**：DevEco Studio 内置体检工具，按官方体验规范做场景化检测（冷启动、时延、功耗等）。
- **AppDeadlineMissed**：应用侧未在 Vsync 期限内完成帧任务产生的丢帧标记（7.1）。
- **AppFreeze**：应用无响应故障的总称，主线程阻塞（THREAD_BLOCK）为典型类型（第 9 章）。
- **appspawn**：应用进程孵化器，负责 fork 应用进程并预置公共初始化（1.4、17.1）。
- **ArkCompiler / 方舟编译器**：ArkTS 的编译与运行时体系，含 abc 字节码与方舟虚拟机。
- **ArkTS**：基于 TypeScript 扩展的鸿蒙应用开发语言，声明式 UI 与并发模型的载体。
- **ArkUI**：鸿蒙声明式 UI 框架，组件树经 Build/Measure/Layout/Render 产出渲染指令（第 2 章）。
- **ArkWeb**：鸿蒙 Web 引擎与 Web 组件体系，独立渲染进程（20.3）。

## B

- **BFCache**：Web 的前进后退缓存，返回页面时不重新加载（20.3）。
- **bm**：设备侧包管理命令，负责 HAP/HSP 安装、卸载、查询（附录 B）。

## C

- **cachedCount**：List/Grid 等懒加载容器的预创建缓存项数量（20.1）。
- **canIUse**：运行时查询设备是否支持某 SysCap 的接口（18.1）。
- **CppCrash**：Native 层崩溃故障及对应日志格式（15.2）。

## D

- **DevEco Profiler**：DevEco Studio 的性能分析器，含 Frame/Launch/CPU/Memory/Snapshot 等会话（第 14 章）。
- **displaySync**：可变帧率接口，允许应用以指定帧率驱动自绘制 UI（20.2、23.1）。
- **DMA_ALLOC**：图片内存分配方式，像素数据驻留 GPU 可访问内存，省掉上传拷贝（20.2）。

## F

- **FaultLog**：系统故障日志，崩溃、AppFreeze 等故障的落盘记录（15.2）。
- **FFRT（Function Flow Runtime）**：系统级并发任务调度运行时，支撑任务依赖编排与调度（5.3）。
- **FrameNode / RSNode**：ArkUI 组件节点与 Render Service 渲染节点，分别在应用侧与 RS 侧表达同一 UI 元素（2.4）。

## G

- **GSKV**：Preferences 的多进程并发存储模式（22.1）。

## H

- **HAP（HarmonyOS Ability Package）**：应用能力包，应用安装与运行的基本单元。
- **HAR（HarmonyOS Archive）**：静态共享包，编译期并入使用方包内，多包引用会产生多份拷贝（23.2）。
- **hb**：OpenHarmony 构建命令行工具，封装 GN 与 Ninja（17.1）。
- **hdc（HarmonyOS Device Connector）**：主机侧设备连接工具，相当于 Android 的 adb（附录 B）。
- **hidumper**：设备侧系统信息导出工具，可查 SA 状态、进程内存与 CPU（附录 B）。
- **hilog / HiLog**：鸿蒙统一日志系统与命令（15.1）。
- **hiAppEvent**：系统事件订阅接口，覆盖崩溃、冻屏、启动耗时、资源泄漏等事件（24.1）。
- **HiTrace**：系统级追踪体系（命令行 hitrace + 打点 API），相当于 Android 的 Perfetto/atrace（13.1）。
- **HiTraceMeter**：应用侧打点 API（startTrace/finishTrace），打点以 `H:` 前缀出现在 Trace 中。
- **HSP（HarmonyOS Shared Package）**：动态共享包，运行时按需加载，多包共享时仅存一份（19.2、23.2）。
- **HWC（Hardware Composer）**：硬件合成器，免 GPU 的图层叠加送显路径，视效交叠可使其失效（2.4、20.2）。

## I

- **init**：OpenHarmony 的 PID 1 进程，读取 init.cfg 拉起系统进程骨架（1.4、17.1）。

## L

- **LazyForEach**：数据懒加载渲染控制，只实例化可视区与缓存区组件（20.1）。
- **Launch 会话**：Profiler 的启动分析场景，把冷启动切成七个量化阶段（14.2、19.1）。

## N

- **NodeContainer**：承载 BuilderNode 自建节点树的容器组件，脱离声明式组件树的手动逃生舱（20.1）。

## O

- **ohpm**：鸿蒙包管理工具，管理 HAR/HSP 依赖（23.2）。
- **OpenHarmony**：鸿蒙的开源基线项目，本书源码引用的出处（1.2）。

## P

- **PixelMap**：解码后的位图像素内存对象，内存治理的重点对象（20.2、21.1）。
- **Preferences**：轻量键值持久化，全量加载内存，适合配置类小数据（6.2、22.1）。

## R

- **RCP（Remote Communication Platform）**：新一代网络请求 API 套件，会话制，支持连接池与响应缓存（22.2）。
- **RDB（Relational Database）**：关系型数据库，默认 WAL 模式、4 读 1 写连接（6.2、22.1）。
- **Render Service（RS）**：独立渲染服务进程，接收应用提交的渲染指令并合成送显（2.4）。
- **renderGroup**：组件离屏绘制缓存开关，重绘优先使用缓存以降低动画绘制负载（20.2）。
- **RenderDeadlineMissed**：RS 侧未按期完成帧的丢帧标记（7.1）。
- **@Reusable**：组件复用装饰器，滑出缓存区的组件进复用池而非析构（20.1）。

## S

- **SAMgr（System Ability Manager）**：系统服务注册与分发中心，按 sa_profile 配置拉起 SA（1.4、17.1）。
- **SA（System Ability）**：系统能力的运行实体，由 SAMgr 管理，可配置随启动加载或按需拉起（17.1）。
- **Sendable**：可跨并发实例共享的对象类型，用引用传递替代序列化拷贝（5.2、22.1）。
- **SmartPerf / SmartPerf-Host**：系统级性能采集工具与主机端分析工具，采集 FPS、CPU、GPU、功耗等（13.3）。
- **Snapshot（堆快照）**：某一时刻 ArkTS 堆的完整镜像，对比两张快照可定位泄漏（14.3、21.1）。
- **Stage 模型**：现行应用模型，以 AbilityStage、UIAbility、ExtensionAbility 组织应用结构（1.3）。
- **startWindowIcon**：启动页图标，在进程创建阶段解码，建议不超过 256×256（19.2）。
- **Structured Clone**：跨线程默认的对象序列化拷贝方式，大对象传递的隐藏开销（22.1）。
- **SysCap（System Capability）**：系统能力集，设备间能力差异的隔离机制（18.1）。

## T

- **TaskPool**：系统管理的并发任务池，适合短时独立任务，线程自动扩缩容（5.1）。

## U

- **UIAbility**：Stage 模型中带界面的应用组件，生命周期含 onCreate/onWindowStageCreate/onForeground 等（1.3、19.1）。

## V

- **Vsync**：显示垂直同步信号，渲染管线的节拍器，一帧周期的起点（2.2）。

## W

- **WAL（Write Ahead Log）**：RDB 默认的日志模式，写操作先记日志再落库（22.1）。
- **Worker**：长驻线程并发单元，适合长时任务与常驻计算（5.1）。

## X

- **XComponent**：自渲染内容容器组件，SURFACE 独立成层、TEXTURE 参与统一绘制（2.5、20.3）。

## 软

- **软总线（DSoftBus）**：分布式设备互联的总线子系统，超级终端的底层通道（1.5）。
