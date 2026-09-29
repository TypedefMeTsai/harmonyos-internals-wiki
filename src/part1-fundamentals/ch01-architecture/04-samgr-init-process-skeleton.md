# 1.4 系统服务框架：SAMgr、init 与进程骨架

## 为什么要有这一层

鸿蒙设备上跑着一百多个系统能力：窗口管理、包管理、电源、传感器、蓝牙、软总线、图形渲染……这些能力由几十个常驻进程承载，进程里又注册了上百个 SystemAbility（SA）。如果没有一个统一的"服务注册与查找"机制，每个应用想调一个系统能力都得知道"它在哪个进程、怎么连"，系统服务之间互相依赖也没法管理启动顺序。

Android 用 ServiceManager + system_server 解决这件事，鸿蒙的对应物是 **SystemAbilityManager（SAMgr）** 加一套 **init 启动框架**。理解它们，才能回答性能排查中反复出现的问题："这个 IPC 为什么慢？""这个系统服务起来了吗？卡在谁身上？"

## 从开机到服务就绪：init 的第一棒

标准系统设备开机后，内核拉起第一个用户态进程 **init**（OpenHarmony 仓库 `base/startup/init`）。init 不实现任何业务，它做三件事：

1. 挂载必要的文件系统、设置 SELinux 等安全上下文；
2. 解析 `/etc/init.cfg` 及各模块的 `.cfg` 配置文件，按配置拉起系统关键进程——每个进程声明了自己的 uid、能力集、onrestart 策略和启动条件（param 触发）；
3. 监护这些进程，异常退出时按策略重启。

init.cfg 里的一个 service 定义大致长这样：

```json
{
  "name": "render_service",
  "path": ["/system/bin/render_service"],
  "uid": "render_service",
  "ondemand": true,
  "secon": "u:r:render_service:s0"
}
```

`ondemand` 字段值得注意：标了它的服务不会开机即起，而是等第一个客户端请求时才被拉起。这是开机耗时优化的重要手段——把"必须立即就绪"的服务缩到最小集合，其余按需启动。[待验证：render_service 在当前商用版本是否仍为 ondemand 配置]

init 拉起的进程里有两个角色要特别记住：**appspawn**（应用进程孵化器，所有应用进程由它 fork 出来）和 **samgr 所在的系统服务骨架进程**。

## SAMgr：系统服务的注册表

SystemAbilityManager（OpenHarmony 仓库 `foundation/systemabilitymgr/samgr`）是整个系统服务层的中枢。它的职责：

- **注册表**：每个 SA 有一个数字 ID（SAID）。比如分布式软总线的 buscenter 是 4700，窗口管理、包管理、AbilityManagerService 各有自己的 SAID。SA 启动后向 SAMgr 注册"我这个 SAID 的服务对象在这里"；
- **查找与路由**：客户端（应用进程或其他服务）拿着 SAID 问 SAMgr 要服务，SAMgr 返回一个 IPC 代理对象，之后的调用直接在客户端与服务之间走，不再经过 SAMgr；
- **按需加载**：SA 的加载策略写在 profile 里（`/system/profile/<saId>.xml`），声明该 SA 由哪个进程承载、是否按需加载、依赖哪些其他 SA。SAMgr 按依赖顺序拉起进程并等待 SA 注册完成；
- **进程组织**：多个相关 SA 可以被配置进同一个进程（比如 foundation 进程承载了一批核心 SA），减少进程数。

[图：SA 调用链路。左侧应用进程内：应用代码 → 框架层 API → IPC proxy（标注 SAID）；中间 SAMgr 进程：维护 SAID → (进程, 服务对象) 的注册表；右侧服务进程：SA 服务对象。时序：① 客户端首次调用时向 SAMgr 查询 SAID（GetSystemAbility）；② SAMgr 若发现 SA 未加载，按 profile 拉起承载进程并等待注册；③ SAMgr 返回代理对象；④ 之后客户端与服务直接 IPC 通信。]

## IPC 骨架：一切都是跨进程调用

应用调一个系统 API（比如 `window.getLastWindow()`），实际发生的是：

1. 应用进程内，框架层把调用打包成 IPC 消息（鸿蒙 IPC/RPC 框架，OpenHarmony 仓库 `communication/ipc`，底层走 binder 驱动）；
2. 消息经 binder 发到服务进程，服务进程的 IPC 线程池接收，解析后调用 SA 实现；
3. 结果原路返回。

单机 IPC 走 binder；跨设备时同一套消息可以被转到分布式软总线（dbinder，见 1.5 节），调用方无感知。这个"单机/跨设备同构"的设计是分布式能力的基础。

对性能分析来说，IPC 有两个要记住的特征。**其一，同步 IPC 会阻塞调用线程直到对端回复**——如果主线程发起了一个同步 IPC 而服务端正忙，主线程就干等，表现出来就是丢帧甚至 AppFreeze（THREAD_BLOCK 类型）。**其二，IPC 数据要序列化**，传大对象（比如大图、长列表数据）的序列化开销随数据量线性增长，该走共享内存（Ashmem/SharedArrayBuffer）的场景别走普通 IPC 参数。

## 实际工作中怎么用

**看系统里有哪些 SA、状态如何：**

```bash
hdc shell "hidumper -ls"                    # 列出所有已注册 SA 的 ID 与名称
hdc shell "hidumper -s 4700 -a '-h'"        # 查看某个 SA 支持哪些 dump 选项
hdc shell "hidumper -s WindowManagerService -a '-a'"   # 导出窗口管理服务全部状态
```

`hidumper -s <SA>` 是系统服务排查的第一入口。官方窗口错误码文档里就用了这条命令确认窗口类型；1.5 节会用 `hidumper -s 4700 -a "buscenter -l remote_device_info"` 看软总线组网状态。每个 SA 都可以实现自己的 dump 逻辑，`-a "-h"` 先问它支持什么。

**开机慢/服务未就绪类问题**：用 `hdc shell "begetctl dump service"` 或查看 init 日志确认目标服务是否被拉起、是否反复重启（onrestart 风暴）。开机阶段某个 ondemand 服务被提前大量请求，也会拖慢整体就绪时间。

**IPC 慢的排查**：Frame/Launch 模板里如果主线程某段空白且没有应用侧 Trace 点，常见原因就是阻塞在 IPC 上。进一步可用 HiTrace 的 IPC 相关 tag 抓对端处理耗时[待验证：IPC trace tag 在各版本的名称]，或者直接看对端进程的 hidumper 状态（线程池是否打满）。

## 版本演进

- OpenHarmony 早期版本 SA 数量少，几乎全部开机即起；
- API 9 之后 SA 数量快速增长，ondemand 按需加载和 SA 依赖编排成为开机优化的常规手段；
- HarmonyOS 5.0+ 的商用设备上，系统服务层进一步拆分了进程（比如把媒体、图形拆成独立进程），单个服务 crash 不再拖垮整组 SA；同时 hidumper 的输出格式逐步规范化，成为官方文档里标准排查手段。

## 常见问题与误区

**"系统 API 调用和普通函数调用一样快"。** 不对。跨进程调用有序列化、线程切换、对端排队三重开销，通常在本机微秒级函数调用的百倍到千倍量级。热路径上（每帧、每次触摸事件）的同步系统调用要格外小心。

**"SA 注册了就一定活着"。** 服务进程会被 OOM 杀掉、会 crash 重启。调用方拿到的代理对象在服务死亡后会收到 DeathRecipient 通知，框架层一般会处理重连，但重连窗口期的调用会失败或阻塞——线上偶发的"系统调用超时"有一类就来自这里。

## 参考资料

- OpenHarmony 仓库：`openharmony/startup_init`（init 启动框架与 init.cfg 解析）
- OpenHarmony 仓库：`openharmony/systemabilitymgr_samgr`（SAMgr 实现，含 SA profile 加载与按需加载）
- OpenHarmony 仓库：`openharmony/communication_ipc`（IPC/RPC 框架）
- 华为开发者文档：hidumper 工具说明（harmonyos-guides/hidumper，`-ls`、`-s`、`-a` 用法）
- 华为开发者文档：错误码 `errorcode-window`（用 `hidumper -s WindowManagerService -a '-a'` 确认窗口类型的实例）
