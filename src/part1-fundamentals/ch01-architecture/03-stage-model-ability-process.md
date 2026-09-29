# 1.3 Stage 模型、Ability 与进程模型

## 为什么需要 Stage 模型

鸿蒙早期的应用模型是 FA 模型（Feature Ability + Particle Ability）。FA 模型把"有界面的组件"和"无界面的服务组件"分成两种 Ability，开发者要在两套生命周期和两套启动方式之间切换；更麻烦的是，FA 模型下每个 Ability 相对独立，组件间共享状态要跨进程或走序列化，复杂应用的内存和启动开销都不好看。

Stage 模型（API 9 起）换了一个思路：以 **Stage（舞台）** 为单位组织应用组件——一个 Module（HAP）就是一个 Stage，Stage 内的多个 UIAbility、ExtensionAbility **共享同一个 ArkTS 引擎实例**，组件之间可以直接共享对象，不再隔着序列化。官方在术语文档里说得直接：Stage 模型支持多个应用组件共享同一个 ArkTS 引擎实例，以及应用组件间的状态共享与对象调用，可以降低内存开销。

这条设计直接解释了三个性能现象：为什么鸿蒙应用的热启动（切回前台）普遍比 Android 快——引擎和进程还活着；为什么同应用内拉起第二个 UIAbility 开销小——不用再起一个 ArkTS 引擎；为什么 UIAbility 之间传大数据不需要 JSON 序列化。

## UIAbility 与生命周期

UIAbility 是带界面的应用组件，继承自 Ability，首批接口从 API version 9 开始支持，仅可在 Stage 模型下使用。它的生命周期围绕四个状态运转：Create、Foreground、Background、Destroy，中间穿插 WindowStage（窗口舞台）的回调。

一次典型的冷启动回调序列是：

```
onCreate → onWindowStageCreate → onForeground
```

切到后台：`onBackground`。再次切回前台：`onNewWant → onForeground`。销毁：`onWindowStageWillDestroy → onWindowStageDestroy → onDestroy`（5.0 之后还补了一批 Will 前缀的回调，如 `onAbilityWillCreate`、`onWindowStageWillRestore`，便于在状态变更前做预处理）。

骨架代码如下：

```typescript
import { AbilityConstant, UIAbility, Want } from '@kit.AbilityKit';
import { window } from '@kit.ArkUI';
import { hilog } from '@kit.PerformanceAnalysisKit';

export default class EntryAbility extends UIAbility {
  onCreate(want: Want, launchParam: AbilityConstant.LaunchParam): void {
    // 应用初始化：只放必要的轻量逻辑，耗时初始化后移
  }

  onWindowStageCreate(windowStage: window.WindowStage): void {
    // 加载 UI 资源。冷启动耗时的主战场
    windowStage.loadContent('pages/Index', (err) => {
      if (err.code) {
        hilog.error(0x0000, 'EntryAbility', 'loadContent failed: %{public}s', JSON.stringify(err));
      }
    });
  }

  onForeground(): void { /* 回到前台，恢复刷新类任务 */ }
  onBackground(): void { /* 退到后台，停掉动画、轮询、定位等 */ }
  onDestroy(): void { /* 实例销毁，释放全局资源 */ }
}
```

这段代码每个鸿蒙开发者都写过，但从性能角度看有三个要点。第一，`onCreate` 里做的事情发生在首帧绘制之前，任何同步耗时（读大文件、同步建库、初始化重型 SDK）都直接计入冷启动时延，应该挪到首帧之后或异步化。第二，`onBackground` 是省电和内存治理的关键回调——官方明确建议在这里注销事件订阅、停止轮询；不做的应用在后台会持续消耗 CPU，也更容易被系统回收。第三，`onWindowStageWillDestroy` 里要注销通过 windowStage 注册的监听器（获焦/失焦、前后台切换），官方生命周期文档专门给了这段注销代码，漏掉就是泄漏。

WindowStage 与 UIAbility 是分离的：UIAbility 管组件生命周期，WindowStage 管窗口。一个 UIAbility 持有一个主窗口，窗口的创建销毁走 `onWindowStageCreate/Destroy` 这条平行线。分析启动 Trace 时，`onCreate` 到 `loadContent` 完成之间的段对应 Launch 模板里的 Ability 阶段，之后才是首帧渲染阶段。

## ExtensionAbility：没有界面的组件

ExtensionAbility 是 Stage 模型下各类"特定场景组件"的统称：卡片（FormExtensionAbility）、输入法（InputMethodExtensionAbility）、分享、备份恢复、DriverExtension 等。它们各自有独立的上下文类型（ExtensionContext）和生命周期，但运行规则与 UIAbility 一致：属于某个 Module，默认与该 Module 同进程（进程规则见下文）。

注意 ExtensionAbility 受系统管控严格：某种类型的 ExtensionAbility 能拉起哪些 Ability 是有限制的，越界调用会拿到明确错误码（`The ExtensionAbility cannot start the ability due to system control`）。排查"拉不起来"类问题先查这类管控，而不是怀疑生命周期。

## module.json5 与 HAP 包结构

Stage 模型的配置中心是 `module.json5`（Module 级）加 `app.json5`（应用级）。一个最小可用的 module.json5：

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "srcEntry": "./ets/entryability/EntryAbility.ets",
    "deviceTypes": ["phone", "tablet"],
    "deliveryWithInstall": true,
    "abilities": [
      {
        "name": "EntryAbility",
        "srcEntry": "./ets/entryability/EntryAbility.ets",
        "exported": true,
        "skills": [{ "entities": ["entity.system.home"], "actions": ["action.system.home"] }]
      }
    ],
    "extensionAbilities": []
  }
}
```

几个字段值得展开。`type` 区分 entry（主模块，随应用安装必有）和 feature（特性模块，可动态交付）；`exported: true` 决定该 Ability 能否被其他应用拉起——被拉时抛 16000004 错误（Cannot start an invisible component），先检查这里；`skills` 声明入口。

打包产物有三种：**HAP**（HarmonyOS Ability Package，含代码与资源的可安装模块）、**HAR**（静态共享包，编译期复用）、**HSP**（动态共享包，运行期共享，多 HAP 共用以减包）。一个应用的安装包（.app）是一到多个 HAP 的组合。包结构与性能的关系：动态交付的 feature HAP 首次使用时要下载，这是"首进某功能慢"的常见原因之一；HSP 能减少包体积但增加一次加载路径，多 HAP 共享大库时值得用。

## 进程模型：默认一个应用一个进程

Stage 模型的进程规则可以概括成一句话：**同一个应用的所有 Module、所有 Ability，默认运行在同一个应用进程里**（一个 ArkTS 引擎实例承载）。这与 FA 模型、也与很多 Android 开发者的直觉（四大组件可随意指定进程）不同。

需要多进程时可以显式声明：在 module.json5 里为 module 或 ability 配置 `process` 标签，相同 process 值的组件进同一进程，不同值则另起进程。典型用法是把 ArkWeb 重度页面或后台媒体模块隔离出去，避免它们 OOM 或 crash 时拖垮主进程。

进程模型的性能含义：

- **内存**：单进程意味着所有页面共享一个堆。某页面泄漏，整个应用 RSS 上涨，Snapshot 分析时要按页面路径过滤对象。
- **崩溃**：任何一处未捕获异常（ArkTS 或 Native）杀掉的是整个进程，所有 UIAbility 一起消失。
- **启动**：进程由 appspawn 孵化（zygote 之于 Android 的对应物），预置了方舟运行时的初始化结果，所以进程创建本身不是冷启动的瓶颈，瓶颈在引擎加载模块字节码和首帧渲染。
- **AbilityStage**：每个 Module 首次加载时创建一个 AbilityStage 实例（Module 级组件管理器），早于该 Module 内所有 Ability 创建，适合做 Module 级初始化；在 onCreate 里还能通过 `application.getApplicationContext().on('abilityLifecycle', ...)` 监听全应用 Ability 生命周期，做统一打点。

## 实际工作中怎么用

**查看应用进程与 Ability 状态：**

```bash
hdc shell "aa dump -l"                    # 查看任务栈里的 Ability 记录
hdc shell "hidumper -s AbilityManagerService -a '-a'"   # AMS 侧全部 Ability 状态
```

**手动拉起与观察生命周期：**

```bash
hdc shell "aa start -b com.example.app -a EntryAbility"
hdc shell hilog | grep -i "ability"
```

**冷启动问题定位入口**：先用 Launch 模板看 onCreate → loadContent → 首帧三段各占多少；onCreate 段长，查同步初始化；loadContent 段长，查模块字节码体积与首帧组件树规模。这在第 19 章会展开成完整案例。

## 版本演进

- API 9：Stage 模型首批接口，与 FA 模型并存。
- API 10–11：WindowStage 回调补齐，Stage 模型成为默认。
- API 12（HarmonyOS 5.0）：FA 模型退出，新应用只支持 Stage 模型；生命周期补一批 Will 前缀回调。
- API 20+：启动框架（AppStartup，startupManager.run）允许把启动任务编排成有依赖关系的任务图，替代在 onCreate 里手排初始化顺序。

## 参考资料

- 华为开发者文档：《UIAbility 组件生命周期》（harmonyos-guides/uiability-lifecycle）
- 华为开发者文档：API 参考 `@ohos.app.ability.uiability`（UIAbility 仅可在 Stage 模型下使用，API 9 起）
- 华为开发者文档：`@ohos.app.ability.abilityStage`（AbilityStage 与 Module 一一对应）
- 华为开发者文档：《Stage 模型开发概述》（stage-model-development-overview）、《应用上下文（Stage 模型）》（application-context-stage）
- 华为开发者文档：《应用术语》（ability-terminology：Stage 模型共享 ArkTS 引擎实例）
- 华为开发者文档：错误码参考 `errorcode-ability`（16000004 与 exported 配置）
- OpenHarmony 仓库：`openharmony/ability_ability_runtime`（Ability 生命周期与进程管理实现）
