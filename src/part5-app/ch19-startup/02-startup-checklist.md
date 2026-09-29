# 19.2 启动优化实战清单

这一节是清单体：每条优化手段给出**为什么有效、怎么做、怎么验证**。验证手段统一三种：Launch 会话（看阶段耗时变化）、AppAnalyzer 场景化体检（看验收口径是否达标）、HiTrace/Frame 泳道（看具体 Trace 点）。引用数据均来自官方《应用冷启动时延优化》最佳实践中的实测对比，测试环境不同数值会有差异，看趋势不看绝对值。

## 1. 生命周期回调里的耗时操作全部挪走

**为什么有效**：AbilityStage.onCreate、UIAbility.onCreate/onWindowStageCreate/onForeground、页面的 aboutToAppear 都在冷启动关键路径上同步执行。官方实测：把一段计算任务从同步执行改为 setTimeout 异步延后后，从 Process Creating 到 First Frame - Render Phase 的耗时从 2.2 秒降到 220.9 毫秒——同一个应用，只差一行调度。

**怎么做**：生命周期回调里只做"不然后面没法跑"的事：创建窗口、注册回调、发起异步任务。其余的一律后移——可以 setTimeout/Promise 延后，可以放 TaskPool，也可以等首帧后再做。判断标准：这段代码不执行完，首帧能不能画出来？能，就挪走。

**怎么验证**：Launch 会话对比挪动前后 UI Ability OnForeground 阶段耗时；用 ArkTS Callstack 泳道确认该函数不再出现在启动阶段的热点里。

## 2. 非启动必需的模块延迟加载（lazy import / 动态 import）

**为什么有效**：冷启动阶段所有被首页 import 链碰到的 .ets 文件都会被加载并执行（包括全局变量初始化）。官方实测：import 模块从 15 个减到 5 个，`H:SourceTextModule::Evaluate` 阶段从 6239.5μs 降到 119.7μs。lazy import 让待加载模块在冷启动阶段不被加载，到实际用到时才按需同步加载。

**怎么做**：

- 对首页不直接使用的模块用 `import lazy { X } from 'path'`。
- 对重模块（大 so、大资源）用动态 import（`import('...')` 返回 Promise）。
- 注意 lazy import 的副作用：它改变模块执行顺序，如果模块顶层有 `globalThis.xxx = ...` 之类的全局赋值，使用方在加载前读该变量会拿到 undefined（见《延迟加载（lazy import）改变模块执行顺序》）。

**怎么验证**：AppAnalyzer 冷启动体检报告里有 import 加载耗时分析——累计加载文件数、未使用文件数、加载总耗时，并能下载依赖关系文件（full_dependency.html）看每个文件的调用链，按耗时从高到低逐个治理。

## 3. 治理 import 写法：按需引用、展开路径、拆 HAR 导出

**为什么有效**：模块加载的耗时由"被执行的文件数量与体积"决定。三组官方实测数据：

- `import * as nm` 全量引用 2000 条数据 vs `import { One }` 按需引用：HandleLaunchAbility 阶段 16.7ms → 7.1ms。
- 8 层嵌套 `export *` vs 直接从目标文件 import：492.6μs → 388.7μs。
- HAR 的 Index.ets 同时导出首页文件和含耗时全局函数的二级页面文件时，首页 `import { MainPage } from 'library/Index'` 会把二级页面文件一起执行。拆分导出文件后，UI Ability Launching 到首帧的阶段从 140.1ms 降到 62.9ms（拆分为 IndexAppStart/IndexOthers 两个导出文件）或 61.3ms（首页 import 时写全路径 `library/src/main/ets/.../MainPage`）。

**怎么做**：

- 工具类文件按需引用具名导出，不用 `import *`。
- 库模块避免多层 `export *` 转出口，首页直接从目标文件路径 import。
- HAR 的导出文件按"冷启动强相关/非强相关"拆成两份，首页只引强相关那份。
- 注意代价：按需引用把变量初始化从冷启动分摊到了后续使用阶段，二级页面首次跳转可能变慢——这是挪耗时不是消灭耗时，全局算总账。

**怎么验证**：Launch 会话观察 abc 加载（`H:JSPandaFileExecutor::ExecuteFromAbcFile`）与 HandleLaunchAbility 阶段耗时；AppAnalyzer 的 import 体检复测。

## 4. 单 HAP 场景用 HAR 不用 HSP

**为什么有效**：HSP 是动态加载的共享包，启动过程中加载每个 HSP 都有额外的 I/O 与运行耗时。官方实测：HAP + 20 个 HSP 混合打包 vs 20 个模块改成 HAR，阶段耗时 34643.7μs → 36.4μs，差了近三个数量级。HAP 和 HSP 都依赖同一批 HAR 时，HAR 会在每个包里各打一份拷贝（这也影响包体积，见 23.2）；但单窗口单 HAP 应用没有"多包共享去重"的需求，多 HAR 直接编进 HAP 是最优解。

**怎么做**：只有需要按需加载（动态下发、运行时加载）的模块才用 HSP；纯代码共享的模块用 HAR。

**怎么验证**：Launch 会话看 Application Launching 阶段的模块加载耗时；数 module.json5 里 shared 类型模块的数量。

## 5. 网络请求提前到 onCreate，首帧数据用缓存预填

**为什么有效**：首页内容依赖网络数据时，请求发起时机决定数据返回时机。官方实测：把网络请求从首页根组件 onAppear 提前到 AbilityStage.onCreate，从启动 Ability 到网络数据返回后首帧刷新的阶段从 1700ms 降到 885.3ms——请求与启动流程并行，而不是等 UI 建完才串行发起。更进一步，上次启动缓存的数据先展示、网络返回后二刷：实测从启动 Ability 到图片显示，641.8ms → 68.9ms。

**怎么做**：

- 首页必需的网络请求在 AbilityStage.onCreate 或 UIAbility.onCreate 里发起（只发起，不 await 阻塞）。
- 响应数据用 AppStorage/状态变量驱动页面刷新。
- 上次的数据落盘（沙箱文件或 Preferences），冷启动先读缓存渲染骨架与旧数据，网络返回后刷新。注意数据时效性：过期数据不适合直接展示的业务不要用。
- 请求本身慢的话上预连接/预解析：启动或空闲时提前完成 DNS 查询与 TCP/TLS 握手，静态资源走 CDN（详见 22.2）。

**怎么验证**：Launch 会话里框选"HandleLaunchAbility 起点 → 网络数据返回后的首个 ReceiveVsync"区间对比；AppAnalyzer 体检报告看"点击离手到请求发起间隔"和请求耗时。

## 6. 主线程不做同步 I/O 与重计算

**为什么有效**：主线程被同步文件读写、大数据反序列化、长计算占住时，首帧渲染排队，直接表现为首帧时延超标。AppAnalyzer 体检会检测"高耗时非 UI 操作"和"异步线程阻塞主线程"（主线程空闲时间越长说明阻塞越久）。

**怎么做**：文件读写、数据库操作、JSON 大对象解析放 TaskPool；跨线程传大数据（100KB 级）用 Sendable 引用传递代替序列化拷贝（详见 22.1）。图片解码下采样、网络数据解析都在子线程完成再回主线程贴数据。

**怎么验证**：Launch/Frame 会话看主线程 Trace 中是否有长段业务函数；体检报告的高耗时函数列表；THREAD_BLOCK 类 AppFreeze 是否消失。

## 7. 启动页图标 startWindowIcon 不超过 256×256

**为什么有效**：启动页图标在进程创建和初始化阶段解码，分辨率过大直接拉长该阶段。官方实测：4096×4096 换成 144×144，启动耗时缩短 37.2ms。Code Linter 有专门规则 `@performance/start-window-icon-check` 扫描这个问题。

**怎么做**：`abilities` 配置里 `startWindowIcon` 指向的资源控制在 256×256 以内；`startWindowBackground` 用纯色而不是大图。

**怎么验证**：Code Linter 扫描；Launch 会话看 Create Process 阶段变化。

## 8. 用 AppStartup 启动框架管理初始化任务

**为什么有效**：当应用有多个初始化任务（SDK 初始化、配置读取、线程预建）且有依赖关系时，手写调度容易全部串行堆在 onCreate 里。AppStartup 框架允许把初始化拆成多个 StartupTask，按依赖关系编排执行，支持自动/手动模式、任务匹配规则和调度阶段设置，并在 AbilityStage 构造过程中开始执行。

**怎么做**：在 `resources/base/profile` 建 startup_config.json 声明任务与依赖，module.json5 的 `appStartup` 字段指过去；每个任务单一职责写成独立 StartupTask 文件；非首帧必需的任务改手动模式在首帧后 `startupManager.run` 触发。

**怎么验证**：Launch 会话看各任务的执行时间点与并行度；确认首帧前只剩必需任务。

## 9. 冷启动并行与预载：TaskPool 预建与页面预加载

**为什么有效**：冷启动阶段 TaskPool 首次扩容、Worker 首次创建都有固定开销；首页之后马上要用的重资源（大列表数据、解码后的图片）如果在首帧期间空闲核上预载，跳转时延直接受益。这是"用启动期的空闲 CPU 换后续交互时延"的思路。

**怎么做**：onCreate 里发起低优先级 TaskPool 任务做数据预热；系统级还有"应用预加载"机制（配置后由系统按用户习惯决定预加载时机，预加载阶段不得包含界面显示与交互操作）。

**怎么验证**：Frame/Launch 会话看预热任务是否落在首帧提交后的空闲窗口；跳转转场时延是否下降（可用 8.2 节的方法度量）。

## 清单的收尾动作

优化做完一轮后，用 AppAnalyzer"手动性能冷启动体检"出正式报告：强行停止应用 → 桌面点击图标 → 结束体检 → 看报告里高耗时函数、import 耗时、网络请求时机、首页组件创建耗时四类结论是否全部转绿。这份报告同时是上架前自测的凭证。

## 参考资料

- 官方文档：《应用冷启动时延优化》（bpta-application-cold-start-optimization）
- 官方文档：《延迟加载（lazy import）改变模块执行顺序》（arkts-module-side-effects）
- 官方文档：《动态加载》（arkts-dynamic-import）
- 官方文档：《AppStartup 应用启动框架》（app-startup）
- 官方文档：《应用启动流程》（application-startup-process）
- 官方文档：《应用预加载》（preload-application）
- 官方文档：《性能分析诊断 AppAnalyzer》（bpta-performance-detection）
- 官方文档：《TaskPool 和 Worker 的对比》（taskpool-vs-worker）
