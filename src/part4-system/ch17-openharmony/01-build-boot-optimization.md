# 17.1 编译构建、镜像裁剪与开机耗时

## 为什么要从构建谈起

做应用性能优化时，我们的起点是"系统已经在那儿"；做系统性能优化时，起点更早——你编译出来的系统本身就是第一个优化对象。一个行业平板的客户不会接受开机 40 秒，一个带屏音箱的 BOM 成本不会给你配 4GB 内存。开机时长、镜像大小、常驻内存基线，这三个指标的答案都埋在两个地方：**你编了哪些东西进镜像**，以及**开机时这些东西以什么顺序被拉起来**。

这一节按动手顺序组织：先理解构建系统怎么描述"编什么"，再学镜像裁剪的方法论，然后量化开机过程，最后用按需加载把开机链路压短。

## OpenHarmony 构建系统：hb、GN 与部件

OpenHarmony 的构建是三层结构：

- **GN（Generate Ninja）**：元构建系统。每个编译单元用 `BUILD.gn` 文件描述目标（executable、shared_library、group 等）及其依赖。GN 解析全部 `BUILD.gn` 后生成 Ninja 构建文件。
- **Ninja**：真正执行编译命令的工具，负责并行调度和增量编译。
- **hb（ohos-build）**：封装在 `build/hb` 目录下的 Python 工具集，是我们日常直接敲的命令。`hb set` 选择产品并生成 `out/{product}/args.gn`，`hb build` 调用 gn + ninja 完成全量或增量编译，`hb clean` 清理输出目录。

日常操作只有三条：

```bash
hb set          # 选择产品（如 rk3568），写入 ohos_config.json
hb build        # 全量编译；hb build -T 部件名 可单独编译某个目标
hb clean        # 清理 out 目录
```

### 子系统、部件与产品

"编什么"由三层配置决定，理解这三层是裁剪的前提：

1. **子系统（subsystem）**：最大的逻辑分组，如 `arkcompiler`、`hiviewdfx`、`distributedschedule`。定义在 `build/subsystem_config.json`。
2. **部件（part）**：子系统内的功能单元，是裁剪的最小粒度。每个部件在自己的源码目录里用 `bundle.json` 声明：自己叫什么、属于哪个子系统、依赖哪些其他部件、编译产物是什么。
3. **产品（product）**：`vendor/{厂商}/{产品名}/config.json` 里列出该产品要包含的子系统及部件清单，同时可以 `inherit` 一个基础产品配置，在其上做加减。

`hb set` 选择的产品决定了最终参与编译的部件集合；GN 再按 `bundle.json` 里的依赖关系补全闭包。这就是为什么"删掉产品配置里的一个部件"是安全的裁剪入口——依赖它的东西如果还在，链接期就会报错暴露问题；它依赖的东西如果只是它独占，就会被连带裁掉。

[图：产品 config.json → 部件闭包 → BUILD.gn → ninja → 镜像文件 的生成链路]

## 镜像裁剪的方法论

裁剪的目标通常有三个：镜像装进更小的 Flash、开机更快（要 mount、要校验的东西变少）、运行内存基线更低（预装服务和库变少）。动手之前先拿到基线：

- 镜像大小：直接看 `out/{product}/packages/phone/images/` 下各 img 文件。
- 文件构成：解包 system.img（`img2simg` + `simg2img` 后挂载，或用 `debugfs`），按目录统计体积。大头通常集中在 `/system/lib64`（共享库）、`/system/app`（预装应用）、`/system/fonts`、`/system/media`。
- 部件映射：把体积大的文件反查到所属部件。`out/{product}/build.config/` 和 `bundle.json` 的 `build` 段能建立"文件 → 目标 → 部件"的映射。

### 裁剪的三个层次

**第一层：产品配置层**。在产品的 `config.json` 中删掉用不到的部件。典型候选：不用电话能力就裁 `telephony` 相关子系统，不用分布式就裁软总线相关部件，不用相机就裁 `camera`。这是收益最大也最安全的层次。

**第二层：特性开关层**。很多部件在 `bundle.json` 里声明了 feature 级开关，通过 GN 的 `declare_args` 暴露，可以在产品配置或 `args.gn` 里覆盖。例如关闭某个图形特性、裁剪 codec 支持列表。这一层裁的是编译进 so 的代码。

**第三层：文件级裁剪**。前两层裁不干净时（比如某个 so 编进去了但产品不需要），可以在产品的 `preloader`/打包脚本阶段从镜像文件列表里剔除，或者修改部件的 `BUILD.gn` 目标依赖。这一层要小心：文件被某个运行中的服务 dlopen 时，裁了就是运行时崩溃，不是编译期报错。

裁剪后必做的验证：全量重编，烧录，跑一遍产品核心场景的冒烟测试，再用 `hdc shell hidumper -s` 确认系统服务列表与预期一致。裁剪引起的最常见故障不是起不来，而是某个 SA 按需拉起时找不到 so，在 log 里留下 `LoadSystemAbility failed` 后静默降级。

## 开机链路：从内核到 Launcher

OpenHarmony 标准系统的开机链路大致是：

```
Bootloader → Linux 内核 → init(PID 1) → ueventd 等基础守护
  → appspawn / samgr 等核心进程（由 init.cfg 拉起）
  → SAMgr 加载系统服务（SA）
  → foundation（含 AMS/BMS/WMS 的载体进程）→ bootanimation → Launcher 首帧
```

每一段的可观测手段不同，先把它们分清楚，后面才能量化。

### init 阶段

OpenHarmony 的 init（`base/startup/init`）是 PID 1。它读取 `/etc/init.cfg`（编译期由各部件的 cfg 片段拼装），按 `jobs` 和 `services` 定义拉起后续一切。两类条目：

- `jobs`：带触发条件（`condition`）的命令集合，例如 `post-init`、`boot` 阶段要做的事情——挂载文件系统、设置系统参数（param）、启动服务。
- `services`：长驻进程定义，包含可执行文件路径、启动条件、是否 critical（挂了是否重启）、socket 声明等。`appspawn`、`samgr`、`hilogd`、`ueventd` 都在这里。

init 阶段耗时主要在文件系统挂载、selinux 策略加载（若启用）和 param 初始化。这一段的日志通过 `hdc shell dmesg` 和 init 写出的 kmsg/日志文件观察，也可以直接用秒表级精度看串口日志时间戳——串口日志每行带时间戳，是开机前 5 秒唯一可靠的数据源。

### 系统参数与 bootevent：开机的"进度条语言"

init 维护一套系统参数（param）服务，用户态通过 `param get/set` 访问（对应命令 `hdc shell param get xxx`）。开机流程用一组 `bootevent.*` 参数表达进度：某个阶段完成时 set 对应的 bootevent，等待方用 param watch 阻塞等待。bootanimation 就是靠等"开机完成"的 bootevent 决定何时退出、把屏幕交给 Launcher。

这给了我们一个低成本的量化方法：在串口或 hilog 里 grep `bootevent`，就能得到各阶段完成的时间点。常见关注点包括 SAMgr 就绪、开机动画启动、boot completed（开机流程结束，Launcher 已可交互）。各 bootevent 的具体名称随版本有调整，以当前基线的 init 与 foundation 源码为准[待验证：6.0 基线 bootevent 参数全量清单]。

### appspawn 与 SAMgr 阶段

**appspawn**（`base/startup/appspawn`）是所有应用进程的孵化器。它自己先完成沙箱、命名空间等公共初始化并进入监听循环；之后每当 AMS 要启动应用进程，就通知 appspawn fork。预加载做得越多（公共库、资源），单次 fork 越快，但 appspawn 自身启动越久、内存基线越高。`appspawn` 的启动在 init.cfg 里，属于开机链路必跑项，它的耗时体现在第一次冷启动应用的"进程创建"阶段——这正是第 19 章讲的 `Create Process` 阶段在系统侧的对应物。

**SAMgr**（`foundation/systemabilitymgr/samgr`）是系统服务的注册与分发中心。它被 init 拉起后，扫描 SA 配置文件（`/system/profile/` 下的 XML，由编译系统从各部件收集），把 `run-on-create` 为 true 的 SA 批量拉起，进入服务循环。SAMgr 就绪后会 set 对应的 bootevent，foundation 等高层进程才能继续。

这一段是开机耗时的大头：几十上百个 SA 的 so 加载、初始化、IPC 注册。用 hitrace 抓开机过程（`hdc shell hitrace -t 30 --trace_begin` 在开机早期执行，或配置开机自启 trace），能看到每个 SA 的加载耗时。

## 系统服务按需加载：sa_profile 的配置艺术

每个 SA 的配置 XML（编译后落在 `/system/profile/`）里，最关键的几个字段：

```xml
<systemability>
    <name>3701</name>
    <libpath>libexample_ability.z.so</libpath>
    <run-on-create>true</run-on-create>
    <distributed>false</distributed>
    <dump-level>1</dump-level>
</systemability>
```

- `run-on-create = true`：SAMgr 启动时立即加载并初始化。开机即需要的服务（AMS、BMS、WMS、电源管理等）必须如此。
- `run-on-create = false`：SAMgr 只登记元数据。当某个客户端首次 `GetSystemAbility` 时，SAMgr 才加载 so、拉起服务。按需 SA 的代价转移到了首次调用方——第一次调用会同步等待 SA 就绪。

按需加载的收益是双份的：开机链路上少拉起一个进程/少加载一个 so，开机时长直接受益；不开机用不到的功能（比如某行业设备永远不用蓝牙），其服务进程根本不出现，常驻内存基线也降。

但按需不是免费的，两个坑：

1. **首调用时延转嫁**。如果一个 SA 是用户操作的即时响应路径上的（比如点亮某个硬件），首次调用的几百毫秒等待会被用户感知到。判断标准：这个服务在"开机后用户前三次操作"里会不会被用到，会，就随开机拉。
2. **依赖级联**。按需 SA A 依赖按需 SA B，首次调用 A 时会级联拉起 B，时延叠加。梳理依赖链，把链路末尾的基础服务保持 `run-on-create`。

验证方法：改动后重编烧录，`hdc shell hidumper -s` 看 SA 列表状态；用 `hdc shell param get bootevent.boot.completed` 配合时间戳对比开机完成时间；用 `hdc shell hitrace -t 20 ability appspawn` 抓开机 trace 对比 SAMgr 阶段耗时。

## 开机耗时优化：一套可执行的流程

把上面的观测手段串起来，一次完整的开机优化流程是：

1. **建基线**。串口日志（内核到 init）+ hitrace（init 之后）+ bootevent 时间点，画出当前开机甘特图。按经验，标准系统参考设备上 init 前内核段约 1-2 秒，init 到 SAMgr ready 约 2-4 秒，SAMgr 到 boot completed 取决于 SA 数量与 Launcher[待验证：参考设备开机各阶段典型耗时]。
2. **找最大段**。哪个阶段占比最大就拆哪个。SAMgr 阶段大 → 清点 `run-on-create` 的 SA 列表；init 阶段大 → 看挂载与 param 服务；bootanimation 退得晚 → 看 Launcher 首帧（这部分回到应用优化方法论，见第 19 章）。
3. **做减法**。能裁的部件裁掉，能按需的 SA 改按需，能延后的初始化延后。
4. **回归验证**。开机时长只是入场券；按需化改多了要回归所有触发 SA 首次拉起的场景，防止出现"用户点了某个冷门功能卡 2 秒"。

## 与应用开发的衔接

如果你同时做应用和系统，这里有一个容易被忽视的配合点：**预装应用的 HAP 放在 system 分区时，首次开机的 dex/abc 处理与签名校验时机**。预装应用是否预置了编译产物、系统首次开机是否要扫描全部预装包，都会影响第一次 boot completed 的时间。行业设备做开机指标时，预装应用的数量和体积要纳入预算，不是系统裁完就完事。

## 参考资料

- OpenHarmony 文档：《编译构建指导》（build 子系统，hb/gn 使用）
- OpenHarmony 文档：《启动恢复子系统》（init、appspawn、bootevent）
- OpenHarmony 源码：`base/startup/init`、`base/startup/appspawn`、`foundation/systemabilitymgr/samgr`、`build/hb`
- OpenHarmony 文档：《系统服务管理子系统》（samgr 与 SA 配置文件）
