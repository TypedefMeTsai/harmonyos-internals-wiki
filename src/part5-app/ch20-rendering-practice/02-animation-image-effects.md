# 20.2 动画、图片与高阶视效

## 动画：先分清你在驱动哪一层

ArkUI 的动画按驱动方式分两类，性能含义完全不同：

- **属性动画/显式动画（animation、animateTo）**：声明起始值和目标值，由系统在每帧插值并更新属性。只改图形变换属性（rotate/translate/scale/opacity）时，不需要重新布局，Render Service 侧直接对 RSNode 的变换矩阵做插值合成，代价最低。
- **帧动画（@AnimatableExtend 逐帧回调、displaySync 自驱、Canvas 逐帧绘制）**：应用侧每帧执行 ArkTS 逻辑计算中间值。每一帧都要走"ArkTS 执行 → 属性更新 → 标脏 → 提交事务"的完整链路，主线程负担重，且帧率上限受主线程空闲程度制约。

官方《帧率与动效选择》最佳实践的案例：用帧动画方式实现的动效实测 63fps（设备支持 120Hz），改用系统属性动画后由系统插值驱动。**默认选择属性动画**；只有属性动画表达不了的（自定义插值路径、物理模拟）才考虑帧动画，并用 displaySync 管理自己的帧节奏。

### 动画优化的五条实操

1. **动画改图形变换属性，不改布局属性**。对组件做位移动画用 `translate`，不要用 `position`/`margin`；缩放用 `scale`，不要改 `width/height`。前者只触发合成层变换，后者每帧都触发 Measure/Layout。
2. **参数相同的动画合并到同一个 animateTo**，多个 animateTo 之间统一更新状态变量——否则下一个 animateTo 执行前又冒出新脏节点，造成冗余更新（《动画丢帧优化》最佳实践）。
3. **大量动效组件开启 renderGroup**。renderGroup 开启后，组件首次绘制时做离屏绘制并缓存，后续重绘优先使用缓存，降低绘制负载。适合"一屏很多个动画小元素"的场景（点赞雨、挂件动效）。代价是缓存占内存，且缓存内容变化时失效重绘——内容静态、变换频繁的组件收益最大。
4. **给动画声明期望帧率**。属性动画和显式动画支持 `expectedFrameRateRange` 参数（expected/min/max），静态场景装饰动画可请求 30Hz；自绘制 UI 用 displaySync 的 ExpectedFrameRateRange 控制刷新节奏。官方帧率建议：小区域动效、微动效建议 30Hz 及以下；插画动效、固定源帧率动效跟随内容源。帧率降下来，功耗和发热直接受益（见 23.1）。
5. **不可见动画必须停**。动画组件滑出屏幕、页面退后台后，displaySync 对象要注销、动画要取消。官方功耗最佳实践里的反面案例：Canvas 弹幕用 displaySync 以 60Hz 刷新，用户关闭弹幕后 displaySync 仍在执行业务，界面静止但帧绘制空跑。

**怎么验证**：Frame 会话看动画期间的帧率曲线与 AppDeadlineMissed；Display Vsync 与 DisplaySync_cb 子泳道看实际绘制帧率是否符合 expectedFrameRateRange；动画结束后确认 Vsync 请求停止。

## 图片：解码、内存与上传的三笔账

图片在渲染链路上有三笔开销：解码（压缩格式 → PixelMap）、内存（PixelMap 驻留）、上传（像素数据进 GPU）。每一笔都有对应的优化手段。

### 解码尺寸匹配显示尺寸

最常见的浪费：3000×4000 的原图解码成完整 PixelMap，显示在 100×100 的头像里。一张 12MP 的图按 RGBA_8888 解码就是约 48MB 像素内存。手段：

- **Image 组件的 sourceSize 属性**：直接指定解码尺寸，让框架按小尺寸解码。
- **解码 API 的 desiredSize**：下采样解码，解码器按最优采样率输出。API 21 起，支持下采样的格式（JPEG/PNG/HEIC）按 1/8 基准梯度（7/8、6/8……1/8）逐次递减取清晰度最高的采样数。JPEG/PNG/HEIF 在解码过程中采用下采样策略，其他格式先完整解码再缩放——所以同样设 desiredSize，JPEG 的收益比 WebP 明显。
- 原图本来就小（图标）时不要设 desiredSize，避免冗余的缩放开销。

### 图片内存与释放

PixelMap 申请像素内存的规则是 `stride × height × 每像素字节数`，图片框架对单张图片解码设了 2GB 内存上限——但进程内存限制远小于这个值，大图不释放就是 OOM 的常客。实践：

- 不使用 PixelMap 时及时释放（页面销毁、列表项回收）。
- 用 `onMemoryLevel` 监听系统内存变化，内存紧张时主动清理图片缓存。
- 大图浏览场景用区域解码（desiredRegion）只解码可视区域，配合下采样做"先缩略图后局部高清"。

### 上传路径：DMA 与纹理压缩

解码后的像素要变成 GPU 纹理。官方《图片 AllocatorType》文档给了明确的对比：使用 SHARE_MEMORY 时 4K 图片单帧渲染耗时约 20ms（CPU 复制到 GPU 显存），改用 DMA_ALLOC 后数据直接保存在 GPU 可访问的内存中，单帧约 4ms。大尺寸图片、高频动态图片加载场景差距显著。

预置资源还有纹理压缩这条路（build-profile.json5 的 resOptions.compression）：官方实测一张图，原图 PNG 加载 62.103ms，纹理超压缩（.sut）15.309ms（4.13 倍收益），ASTC 38.239ms（1.63 倍）。资源目录里的启动图、引导页大图适合开。

### 列表图片：懒加载与缓存组件

列表中的网络图片：Item 进入可视区才加载（LazyForEach 天然支持）、placeholder 先行、加载完成后淡入。图片框架自带缓存不可定制、无法观测命中率，复杂业务用 ImageKnife（三方图像库）拿到可配置的内存/磁盘缓存策略和生命周期管理。下载侧的经验：单张网络图超过 10MB 或一次下载数量较多时，用 HTTP 工具提前下载落盘再加载，比让 Image 组件直接拉网络流可控。

## 高阶视效：离屏渲染与 HWC 失效

模糊（backgroundBlurStyle、foregroundBlurStyle）、反色、提亮、线性渐变模糊这类视效，在渲染管线里意味着**离屏渲染**：先把背景或内容画到一块离屏缓冲，执行模糊/滤镜算法，再合成回来。代价分三层：

1. **每帧实时计算**。backgroundBlurStyle 这类是实时模糊接口，每帧都执行实时渲染，性能负载大。官方建议：模糊内容和模糊半径都不变时，用静态模糊接口 blur（一次计算）。
2. **取色方式差价**。同样是背景模糊，用 ColorPicker 取色绘制的单帧平均耗时 5.650ms，直接设 AdaptiveColor.Average 参数是 9.400ms（《高效使用模糊》文档实测）——视效参数的选择本身就是性能决策。
3. **HWC 失效**。这是视效最隐蔽的代价。正常情况下 Render Service 把图层交给 HWC（硬件合成器）直接叠加送显，GPU 不参与最终合成，省电省带宽。但**当自渲染图层（视频、XComponent、Web 的 surface）上方的 UI 控件使用模糊等高阶视效时，模糊算法需要实时读取背景图层内容并做额外处理，HWC 无法使能，RS 只能改用 GPU 合成**（《高效使能 HWC》最佳实践）。视频页面上叠一个模糊底衬的弹幕栏，就是典型的"看着很美、HWC 失效、发热上来"的组合。

视效使用的工程原则：

- 视觉参数一致的多个组件，合并视效（一次模糊算一组，而不是每个组件各算一次）。
- 沉浸式系统材质自带背景模糊时，不要再叠 backgroundBlurStyle/backgroundEffect，重复处理是纯浪费。
- 视频、游戏等自渲染图层附近，视效能不加就不加，必须加时挪开交叠区域（详见 20.3）。
- 图层按需创建，过多图层会导致硬件叠加功能失效，系统要多付出功耗开销（《应用性能体验建议》合成使用建议）。

**怎么验证**：Frame 会话看 RenderFrame（GPU 绘制）耗时变化；用 hidumper 的图形模块或 Profiler 的图层信息看 HWC 是否使能（对比加/去视效前后的合成方式）；功耗侧用 23.1 的方法看 GPU 负载与温升。

## 参考资料

- 官方文档：《动画丢帧优化》（bpta-animation-frame）
- 官方文档：《帧率与动效选择》（bpta-zhenlv）
- 官方文档：《DisplaySync 可变帧率》（js-apis-graphics-displaysync、displaysync-animation）
- 官方文档：《Vsync 功耗优化》（bpta-vsync-power-optimization）
- 官方文档：《图片显示》（arkts-graphics-display）
- 官方文档：《图片 AllocatorType》（image-allocator-type）
- 官方文档：《区域解码与下采样》（image-region-and-downsampling）
- 官方文档：《纹理压缩提升性能》（bpta-texture-compression-improve-performance）
- 官方文档：《高效使用模糊》（ui-use-blur-efficiently）、《模糊效果》（arkts-blur-effect）
- 官方文档：《高效使能 HWC》（bpta-utilize-hwc-efficiently）
- 官方文档：《应用性能体验建议》前台渲染章节
