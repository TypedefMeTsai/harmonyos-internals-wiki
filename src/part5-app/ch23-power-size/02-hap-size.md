# 23.2 HAP 包体积治理

## 先解包，再优化

包体积治理的第一步永远是测量：HAP 本质是 zip，解开后按目录统计体积，大头通常落在四处——`libs`（各 ABI 的 so）、`resources`（图片/媒体）、`ets`（编译后的字节码）、第三方库。没有这张体积分布图，后面所有动作都是猜。

## 资源精简：通常的最大头

- **删未使用资源**： drawable、media 目录里不再引用的图片直接删；DevEco 的资源引用检查能找出无引用资源。
- **图片格式与压缩**：能用矢量（SVG）的不用位图；大图评估转 WebP 等高效格式；预置大图可开纹理压缩（resOptions.compression），同时还能吃加载速度收益（20.2 引用的数据：PNG 62.1ms → SUT 15.3ms）。
- **分辨率适配审计**：只保留目标设备真正用到的分辨率档位，为不存在的屏幕密度备的图是白占体积。
- **按需资源**：非首屏必需的大资源（高清素材、引导视频）评估改为云端下发。

## so 裁剪：架构、重复与压缩

Native 库是第二个大头，三个动作：

1. **裁剪 ABI**：产品只发 64 位设备就别打 armeabi-v7a 的 so；确认 ohpm 三方库没有偷偷带进多余架构。
2. **去重**：build-profile.json5（app 级）提供 `deduplicateSo` 配置——构建 APP 时去除 HAP 和 HSP 中重复的 so 文件，so 只保留一份并随机打包在某个 HAP/HSP 中（hvigor 26.0.0 起支持，OpenHarmony 工程不支持）。多包工程必开。
3. **压缩**：module.json5 的 `compressNativeLibs` 或 hvigor-config.json5 的 `ohos.pack.compressLevel` 开启 so 压缩。so 数量多时压缩耗时可观，DevEco Studio 6.1.0 Beta1 起可开 `enableIncrementalSoCompress`：so 内容未变时复用上次压缩结果，加快增量打包。

## ohpm 依赖治理

三方库依赖是体积雪球的主要来源。治理动作：

- `ohpm list -r` 分析工程依赖链路，找出"为了一个小函数引进一个大库"的依赖；
- 同一功能的多份实现去重（两个库都带了 JSON 解析、都带了图片加载）；
- 检查传递依赖的版本重复——同一库被不同依赖拉了多个版本时，评估对齐到单一版本；
- 不更新的死库及时移除。

## 混淆与压缩

ArkGuard 混淆同时是安全手段和体积手段：名称混淆缩短标识符，`-compact` 做代码压缩，`-remove-log` 删除日志代码、`-remove-nosideeffects-calls` 删除无副作用调用。官方建议 HAP 包开启推荐的四项混淆规则，其它按需添加。注意边界：ArkGuard 只处理 ArkTS/TS/JS 代码，不管 C/C++、JSON 与资源；混淆只在 release 编译生效，验证行为差异时用"同一构建模式开/关混淆"对比，不要靠切 debug/release 区分（两者差异不止混淆）。

字节码混淆有独立的一套配置（obfuscation-rules.txt 等三个配置文件），HSP 包独立构建且只构建一次——HSP 开发者只应配白名单保留规则（-keep-global-name、-keep-property-name），开启类配置交给使用方，避免混淆规则互相污染。

## 分包选型：HSP 与 HAR 的体积账

多包场景的核心体积问题：**多个 HAP/HSP 引用同一个 HAR 时，打包后每个包里都有一份该 HAR 的拷贝**。官方《减小应用包体积》最佳实践的结论：改用 HSP 承载共享代码与资源，APP 包中只保留一份拷贝——当共享 HAR 的总大小超过 HSP 自身的开销时，用 HSP 代替 HAR 能减小应用包体积。

但这不是单向结论，和第 19 章形成一组对照：

- **体积视角**：多 HAP 共享大模块 → HSP 去重。
- **启动视角**：单 HAP 应用的多模块 → HAR 更优（HSP 动态加载增加启动 I/O 与耗时，20 个 HSP vs 等效 HAR 的启动阶段耗时是 34643.7μs vs 36.4μs）。

决策规则：单 HAP 用 HAR；多 HAP 且共享模块大 → 公共部分提 HSP；HSP 之间、HAP 与 HSP 之间依赖相同 HAR 且该 HAR 不需要按需加载时，按《多 HAP/HSP 构建实践》的思路自顶向下改造（`ohpm list -r` 看依赖链，把无按需加载需求的 HSP 改造为 HAR，降低编译耗时、内存占用和包体积）。

另外**按需加载**本身是体积策略：官方支持把应用分段，用户先下载基本功能包，增强功能包在使用时再从服务器下载——对功能庞大但用户路径集中的应用（工具箱类），这能把初始下载体积砍掉一半以上。

## 治理流程

1. 解包出体积分布图，列各目录预算；
2. 按"资源 → so → 依赖 → 代码"的顺序逐项治理，每项改完重打包对比；
3. 体积指标纳入版本回归：每次发版对比 HAP/APP 体积，异常增长（比如 +20%）必须找到来源才能放行。

## 常见误区

**"混淆能显著减体积"**：混淆对代码体积的压缩是百分比有限的（名称缩短 + 日志删除），它首先是安全手段。期望混淆把 100MB 的包砍半会落空——大头永远在资源和 so。

**"资源删了就没事了"**：删资源要查全引用链，包括字符串拼接路径、rawfile 动态加载、服务端下发的资源名约定。静态扫描找不到的引用，删完就是运行时白图。

**"HSP 越多架构越清晰"**：每个 HSP 都是独立的安装/加载单元，启动期要逐个加载（19.2 的 34643.7μs vs 36.4μs 对比就是 20 个 HSP 的代价）。架构分层是代码组织问题，用 HAR 目录结构就能解决，不要把分层冲动变成运行时成本。

**"按需加载适合所有大功能"**：功能分段的代价是"首次使用时下载"，用户在网络差的场景点进未下载功能是负体验。适合分段的是使用率低、用户路径明确的功能（年度账单、深度设置项），核心路径功能必须随主包。

## 参考资料

- 官方文档：《减小应用包体积》（bpta-decrease_pakage_size）
- 官方文档：《多 HAP/HSP 构建实践》（ide-multi-hap-hsp-practice）
- 官方文档：build-profile.json5 配置参考（deduplicateSo）
- 官方文档：《hvigor 实验性配置》（ide-hvigor-experimental-properties，enableIncrementalSoCompress）
- 官方文档：《ArkGuard 混淆开启指南》与《混淆最佳实践》（source-obfuscation、source-obfuscation-practice）
- 官方文档：《字节码混淆指南》（bytecode-obfuscation-guide）
- 官方文档：《纹理压缩提升性能》（bpta-texture-compression-improve-performance）
