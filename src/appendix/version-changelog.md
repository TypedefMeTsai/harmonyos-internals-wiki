# 附录 A：HarmonyOS 版本与 API 速查

## 版本对照表

| 版本 | API Level | 发布时间 | 说明 |
| --- | --- | --- | --- |
| HarmonyOS 4.x | API 9-11 | 2023-2024 | 双框架时代末期，兼容 AOSP，支持 APK 与 HAP |
| HarmonyOS NEXT / 5.0 | API 12 | 2024-11 | 移除 AOSP 兼容层，仅支持 HAP；纯鸿蒙生态起点 |
| HarmonyOS 5.1 | API 14 | 2025 年中 | 随 Pura 80 系列发布 |
| HarmonyOS 6.0 | API 20 | 2025-11 | 不再带 NEXT 后缀；对应 OpenHarmony 6.0 开源基线 |
| HarmonyOS 6.1 | API 21 | 2026 | 随 Pura 90 系列发布 |

OpenHarmony 开源主干与商用版本的 API Level 对齐：OpenHarmony 6.0 即 API 20 的开源基线，本书机制源码引用以此为准。双框架时代（4.x 及更早）不在正文覆盖范围，此处仅作历史对照。

## 各版本与性能相关的关键变化

### 双框架时代简述（HarmonyOS 4.x，API 9-11）

- ArkUI 声明式范式成为主推，FA 模型向 Stage 模型迁移完成。
- 性能工具链以 DevEco Studio Profiler + hilog + hidumper 为主，SmartPerf 开始在社区可用。
- 兼容层存在意味着同一设备上 ArkUI 应用与 AOSP 应用并存，性能分析要先分清目标进程跑在哪个栈上。

### HarmonyOS NEXT / 5.0（API 12）

- 移除 AOSP 兼容层：系统服务、渲染、应用框架全面纯化，Trace 中不再出现 Android 系进程，泳道结构即本书第 13 章描述的形态。
- Stage 模型唯一化，AbilityStage/UIAbility 生命周期成为启动分析的统一框架（第 19 章）。
- 折叠屏高级组件就位：FoldSplitContainer、FolderStack（API 12 起）。
- RDB 的 Sendable 支持（SendableRelationalStore）从 API 12 提供。
- hiAppEvent 系统事件订阅体系（崩溃、冻屏）成熟，线上质量体系的端侧基础。
- Web 同层渲染、Native 媒体接管等 ArkWeb 高级能力陆续开放。

### HarmonyOS 5.1（API 14）

- 随 Pura 80 系列发布，主要面向新机型的硬件能力开放（影像、显示），性能框架无结构性变化。

### HarmonyOS 6.0（API 20）

- 命名上不再带 NEXT 后缀，生态进入常规演进期。
- `aa start -W` 调优参数（API 20 起）：命令行直接测量 UIAbility 冷启动到首帧绘制的 TotalTime，无 Profiler 环境的快速测量手段。
- Test Kit 提供白盒性能自动化测试能力（API 20 起）：启动时延、页面切换时延、列表滑动帧率的场景化采集。
- 元服务多设备展示效果的开发者自主控制字段（deviceType、resizable 等）从 API 20 起支持。
- startupManager.run 支持指定 AbilityStageContext 执行启动任务（API 20）。

### HarmonyOS 6.1（API 21）

- 随 Pura 90 系列发布。
- 图片下采样解码规则明确化（API 21 起）：设置 desiredSize 后按 1/8 基准梯度逐次递减取最优采样率，支持下采样的格式以设备规格为准。
- List 单个子组件最大宽高从 1000000px 放宽到 16777216px（API 21 起）。
- 元服务 API 的折叠屏组件（FolderStack 等）支持范围扩大。

## 版本相关的性能分析注意事项

- **Trace 点名称随版本演进**：本书引用的 Trace 点（如 H:RSMainThread::ProcessCommandUni）以 API 20 基线为准，新版本可能改名或拆分，分析时以当前设备 `hitrace -l` 与 Profiler 实际泳道为准。
- **体验规范数值按版本更新**：≤1100ms 启动时延等数值来自当前《应用性能体验建议》，历史版本的口径可能不同，上架检测以当期规范为准。
- **API 可用性判断**：多版本适配时用 canIUse 做能力判断，不要按版本号硬编码分支。
