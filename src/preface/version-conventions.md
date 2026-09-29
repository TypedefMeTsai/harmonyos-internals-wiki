# 版本约定

## 版本基线

| 名称 | 对应关系 | 说明 |
| --- | --- | --- |
| HarmonyOS NEXT | HarmonyOS 5.0，API 12 起 | 2024-11 发布，移除 AOSP 兼容层，仅支持 HAP |
| HarmonyOS 5.1 | API 14 | 2025 年中随 Pura 80 系列发布 |
| HarmonyOS 6.0 | API 20 | 2025-11 正式发布，不再带 NEXT 后缀 |
| HarmonyOS 6.1 | API 21 | 2026 年随 Pura 90 系列发布 |
| OpenHarmony 6.0 | API 20 对应的开源基线 | 开源主干，本书机制源码引用以此为准 |

双框架时代（HarmonyOS 4.x 及更早，兼容 AOSP）不在正文覆盖范围内；历史差异只在"版本演进"小节中简述。

## 引用格式

- 官方文档：`[官方文档：文档名]`，指 developer.huawei.com 上的开发指南 / 最佳实践。
- OpenHarmony 源码：`仓库名/文件路径`，如 `graphic_graphic_2d/rosen/modules/render_service/core/pipeline/rs_main_thread.cpp`。
- API 标注：`API 12+` 表示自 API 12 起可用或行为生效。

## 验证状态标注

- 无标注：来自官方文档或开源代码，可核对。
- `[待验证：...]`：作者无法实测确认，读者遇到时应以当前设备实测为准。
- `[图：...]`：待补充的图或 Trace 截图描述。
