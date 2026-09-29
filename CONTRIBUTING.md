# 贡献指南

欢迎提 Issue 和 Pull Request。

这本书的机制来自华为开发者官方文档、OpenHarmony 开源代码和公开资料，作者做的是按 HarmonyOS 6.x / API 20+ 核对、整理和写清适用范围。写错了请直接指出。合并由维护者人工完成。

## 参与方式

两条路就够：

1. **Issue**：报错、提问、建议补哪一章。
2. **Pull Request**：直接改正文。适合错别字、失效链接、事实勘误、补例子或观测方法。

新章节、整节重写、大段案例，先开 Issue 说清楚要补什么、依据是什么，讨论后再写 PR。

## 开 Issue 时请写清

- 章节号或文件路径，例如 `2.2` 或 `src/part1-fundamentals/ch02-rendering/02-vsync-unified-rendering.md`
- 现在怎么写、你认为应该怎么写
- **HarmonyOS 版本与 API Level**。正文结论基线是 HarmonyOS 6.x / API 20+，机制源码引用 OpenHarmony 6.0 主干。更低版本或更新预览版的材料不要当成当前结论
- 官方文档 URL、OpenHarmony 仓库路径，或可复现的观察方法
- 期望现象和实际现象（如果是排障类问题）

缺少版本和依据时，维护者会先请你补，再往下处理。

## 开 Pull Request 时

- 从 `main` 拉分支，改动尽量小，一个 PR 只做一件事。
- 正文在 `src/` 下，目录以 [`src/SUMMARY.md`](src/SUMMARY.md) 为准。新增文件要同步加进 SUMMARY。
- 提交说明写人话即可，例如 `fix: 2.2 更正 Vsync 周期在 60Hz 下的数值`。
- 写作规范见 [`writing-guide.md`](writing-guide.md)，尤其是文风铁律一节：不用赋能、闭环、底座、抓手这类词，不写"综上所述""显然""众所周知"。
- 许可见 [LICENSE](LICENSE) 和 [COMMERCIAL-LICENSE.md](COMMERCIAL-LICENSE.md)。提交 Pull Request 即表示：
  1. 你将贡献按 CC BY-NC-SA 4.0 授权给公众（署名、非商业、相同方式共享）；
  2. 你同时授予版权人 TypedefMeTsai 永久性、全球范围的许可，可将该贡献用于本项目的所有版本，包括商业出版、再许可和修改后的版本。
  你保证贡献是自己的原创，或你有权按上述条件授权。

小的事实修正可以直接 PR。拿不准就先 Issue。

## 正文要求

- 技术结论能回到源码、官方文档或可复现观察。不要补不存在的 API、类、路径、Trace 点名或时延数值。
- 引用官方数据时注明出处；无法核实的内容标注 `[待验证：...]`，不要猜测。
- 不要整篇搬运官方文档或他人文章。可以提取事实，用自己的话重写，并标明出处。
- 需要配图的地方用 `[图：{描述}]` 占位，不要贴无法确认来源的截图。
- 术语统一：用 Render Service、RSNode、TaskPool、Sendable、HiTrace、AppFreeze 这些官方原名，不要自造中文译名。

## 事实来源优先级

1. 华为开发者官方文档（developer.huawei.com）
2. OpenHarmony 开源代码（gitee.com/openharmony）
3. 公开的版本说明与 API 参考
4. 社区实践（需标注并说明适用范围）

## 致谢

合入的贡献者会记在 README 里。
