# 1.2 OpenHarmony 与 HarmonyOS 的关系

## 为什么要在乎这两个名字

做鸿蒙性能分析的人迟早会去翻源码。翻到一半经常会冒出两个问题：Gitee 上这套 OpenHarmony 代码和我手里这台华为手机，到底是不是同一个系统？为什么有的机制源码里写得清清楚楚，有的却找不到对应实现？

回答这两个问题，需要先理清 OpenHarmony 与 HarmonyOS 的分工，以及 API 版本演进的脉络。否则我们会犯两类错误：拿开源代码的行为去断言商用机的行为，或者反过来，因为商用机不开源就放弃了对机制的理解。

## OpenHarmony 是开源项目，HarmonyOS 是商用发行版

**OpenHarmony** 是华为把鸿蒙的基础能力开源后、由开放原子开源基金会孵化的开源项目，代码托管在 Gitee（`gitee.com/openharmony`）。它包含内核抽象层、系统服务框架、ArkUI、方舟运行时、分布式软总线等完整系统栈，可以被任何厂商拿去做自己的设备——开发板、行业平板、智能家居主机，都有基于 OpenHarmony 的商用产品。

**HarmonyOS**（商用手机/平板上的那个）是华为基于 OpenHarmony 做的商用发行版。关系类似 AOSP 与 Pixel 手机上的 Android：主干同源，但发行版里包含不开源的部分——华为的 HMS 服务、部分图形与媒体增强、安全组件、应用市场等。

这个关系对技术学习有三层含义：

1. **系统运行机制可以对着 OpenHarmony 源码学。** ArkUI 渲染管线（`openharmony/arkui_ace_engine`）、Render Service（`openharmony/graphic_graphic_2d`）、软总线（`openharmony/communication_dsoftbus`）、TaskPool 与 Sendable（`openharmony/arkcompiler_ets_runtime`）这些本书反复引用的模块，在 OpenHarmony 里都有完整实现，机制描述与商用 HarmonyOS 一致。
2. **涉及华为自有服务的部分只能看官方文档。** 应用市场、账号体系、推送、部分性能调度策略，源码不可见，只能以 developer.huawei.com 的文档和实际抓取的 Trace 为准。
3. **版本号要分开看。** OpenHarmony 有自己的版本线（OpenHarmony 4.x/5.x/6.x），商用 HarmonyOS 用另一套版本线（HarmonyOS 4、5、6），两者通过 API 版本号对应到应用开发侧。

## API 版本演进：从 API 9 到 API 20+

应用开发侧真正要面对的是 API 版本。梳理一下与本书相关的里程碑：

| API 版本 | 对应系统 | 关键变化 |
| --- | --- | --- |
| API 9 | HarmonyOS 3.1（双框架时代） | Stage 模型首批接口，UIAbility、module.json5 登场 |
| API 10/11 | HarmonyOS 4.x | Stage 模型成熟，@Reusable（API 10）等性能能力补齐 |
| API 12 | HarmonyOS NEXT / 5.0 | 单框架起点，状态管理 V2（@ObservedV2/@Trace）首批支持 |
| API 13–19 | HarmonyOS 5.x | XComponent、并发、图形能力持续增强；5.1 起元服务 API 集扩大 |
| API 20/21 | HarmonyOS 6.0/6.0.1 | 应用启动框架（startupManager，API 20）等 |
| API 23+ / 26 | HarmonyOS 6.1 及之后 | @SyncMonitor（API 23）等；版本号格式在 API 24 之后切换为与语义化版本一致的新格式（如 26.0.0 即 API 26），见 `deviceInfo.distributionOSApiVersion` 文档 |

几个容易踩的坑：

- **API 12 是分水岭。** 不只是"单框架"的商业变化，很多性能相关能力（状态管理 V2、新的帧率工具链）都以 API 12 为起点。写兼容性代码时，API 12 前后经常要分开处理。
- **接口的"起始版本"以文档标注为准。** 官方 API 参考里每个接口都标了"从 API version X 开始支持"，元服务还有单独的 API 集约束。引用本书机制描述时，同样要注意适用范围。
- **同一 API 版本在不同设备形态上的能力不同。** API 版本号之外还有 SysCap 维度（见 1.1 节），版本号够不代表能力在。

## 双框架时代到单框架：一段必须了解的历史

2023 年之前的商用 HarmonyOS（2.0 到 4.x）在手机上是"双框架"：系统里同时存在鸿蒙自有框架和 AOSP 兼容层，因此能安装运行 Android APK。这也是早期"鸿蒙是不是安卓套壳"争论的技术来源——兼容层确实来自 AOSP，但鸿蒙自有框架（Ability、ArkUI、方舟运行时）在同一台机器上独立运行。

双框架的代价很实际：一套系统维护两套应用框架，内存、功耗、启动速度都背着包袱；而且应用在两套框架之间的边界行为（比如 ArkUI 页面嵌入 WebView 加载 H5 再调原生）复杂且难以优化。

2023 年下半年华为发布 HarmonyOS NEXT 开发者预览版，2024 年随 5.0（API 12）商用，**彻底移除 AOSP 兼容层，进入单框架时代**：只运行鸿蒙原生应用（.app 包，内含 HAP），APK 无法安装。对性能工程师来说，这意味着：

- 性能分析不再需要考虑"应用跑在哪套框架上"的歧义；
- 冷启动链路、渲染链路、并发模型全部收束到 ArkTS/ArkUI/方舟运行时一条技术线上，本书第一部分讲的就是这条线；
- 存量 Android 应用的迁移质量（尤其是用跨平台框架仓促迁移的）成为新的性能问题来源——第五部分的不少案例与此有关。

## 实际工作中怎么用

**查机制：先 OpenHarmony 源码，后官方文档，最后实测。** 比如想确认 ArkUI 一帧的处理流程，先看 `openharmony/arkui_ace_engine` 的 `frameworks/core/pipeline_ng/pipeline_context.cpp`（`FlushVsync`），再用官方帧率分析文档校准术语，最后在真机上抓 Frame 模板验证。三步走完的结论才可靠。

**确认版本与能力：**

```bash
hdc shell "param get const.ohos.apiversion"
hdc shell "param get const.product.software.version"
```

前者是当前设备的 API 版本，后者是商用版本号。报障或复现问题时，这两个值必须记录——本书所有版本相关结论都标注了适用范围，排查时请先对照。

**读源码时留意商用差异。** OpenHarmony 默认产品的调度参数、图形配置与华为旗舰机不完全一样（比如 Vsync 分发策略、内存水位）。源码告诉我们"机制是什么"，具体参数以真机实测为准，本书会在这类地方标注 [待验证]。

## 常见问题与误区

**"看了 OpenHarmony 就懂 HarmonyOS"。** 机制层面成立，策略层面不成立。商用机有大量不开源的调度与管控策略（后台管控、性能模式），只能靠 Trace 和官方文档反推。

**"单框架之后 Android 经验没用了"。** 恰恰相反，Vsync 驱动的渲染模型、binder 风格的 IPC、低内存回收这些概念在鸿蒙里都有对应物，Android 性能工程师的直觉大部分可迁移，本书会在对应小节做一次性类比。需要丢弃的是具体 API 和工具名，不是分析方法论。

## 参考资料

- OpenHarmony 项目主页与仓库列表（gitee.com/openharmony）：`arkui_ace_engine`、`graphic_graphic_2d`、`communication_dsoftbus`、`arkcompiler_ets_runtime`
- 华为开发者文档：《所有 HarmonyOS 开发套件版本》与 `js-apis-device-info`（API 版本号与 distributionOSApiVersion 格式说明）
- 华为开发者文档：HarmonyOS 5.0/6.0 版本说明（guidebook solution2/solution3 章节）
- 华为开发者文档：《元服务 API 集》说明（应用与元服务能力差异）
