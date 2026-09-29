# 2.4 Render Service、RSNode 与 HWC 送显

## 为什么系统侧这一半必须懂

应用在统一渲染架构下只画"指令"，真正把像素算出来的是 render_service 进程。这个进程出问题时有两个特征：一是影响面大——所有窗口一起卡；二是应用怎么优化代码都没用——瓶颈不在自己进程里。还有一种更隐蔽的情况：帧率完全正常，但功耗高、机身热，根因在送显路径上——本该走硬件直通的图层被拖去 GPU 重绘了。这节把 Render Service 的内部结构和两条送显路径讲清楚。

## Render Service 进程结构

render_service（OpenHarmony 仓库 `openharmony/graphic_graphic_2d`）是系统服务层的图形进程，核心结构：

- **RSNode 树**：应用侧每个 FrameNode 的 RenderContext 在这边有一个对应的 RSNode。应用发来的事务（`MarshRSTransactionData`）更新这些节点的属性——位置、透明度、裁剪、模糊、子树结构。
- **RSMainThread**：接收并处理各应用进程的事务命令（Trace 里 `RSMainThread::ProcessCommandUni`），维护全局 RSNode 树和屏幕（RSDisplayNode）状态。
- **RSUniRenderThread**：统一渲染线程，Vsync 到来时遍历可见窗口的 RSNode 树，生成 Skia 绘制命令并提交 GPU，完成各图层的绘制与合成（Trace 里 `RenderFrame` 段）。
- **DirtyRegion 脏区管理**：RS 不整屏重绘。每个节点属性变化会把受影响区域标进脏区，绘制时只处理脏区覆盖的部分。界面只有一个小区域动（比如一个时钟秒针），GPU 工作量远小于整屏——这是为什么"局部小动画"本身不怎么费电，而"全屏背景模糊"很费。

帧的尾部是送显：合成结果经 `PresentFence` 等 Trace 点交给显示子系统。帧是否准点，Frame 模板 render_service 泳道一目了然（2.2 节的排查看法）。

## 两条送显路径：GPU 合成 vs HWC 直通

屏幕上的最终画面由若干图层（layer）叠加而成：应用主 UI 图层、视频/XComponent 的自渲染图层、状态栏、弹窗等。叠加方式有两种：

**GPU 合成（Client Composition）**：RSUniRenderThread 把所有图层当纹理，在 GPU 里画成一张完整帧再送显。灵活——任何图层可以做任意变换、模糊、混合——但 GPU 每帧都要为全屏工作。

**HWC 合成（硬件叠加/直通）**：硬件合成器（Hardware Composer）有若干硬件图层通道，可以在显示时直接把多个 buffer 按位置叠加输出，不经过 GPU 重绘。自渲染内容（视频解码输出的 buffer、XComponent 里游戏引擎画的 buffer）走这条路最划算：buffer 从生产者直接进 HWC，RS 只做图层管理不做像素工作。官方 Buffer 功耗优化文档描述了理想的直通形态：自绘制图层获取 Buffer、释放 Buffer，无需交由 RSUniRender 重绘计算，也没有 DisplayNode 的刷新显示。

HWC 的收益是实打实的功耗。官方《高效利用 HWC 的低功耗设计》给过一组实测：**Web 组件上方控件去除模糊效果后，总功耗下降约 25.5%，CPU 模块功耗下降约 30.7%**——一个模糊控件，就把整个 Web 自渲染图层从 HWC 直通打回 GPU 合成，整机功耗应声而涨。

## 什么会破坏 HWC 直通

规则一句话：**当一个自渲染图层需要参与任何像素级计算时，它就不能直通了。** 官方文档列出的"不推荐用在自绘制图层上"的效果包括：

- `opacity` 给自绘制组件设置透明度——图层要和下方内容做 alpha 混合，必须 GPU 参与；
- `foregroundEffect` 前景模糊、`blur` 背景模糊——模糊要采样周边像素；
- `grayscale` 灰阶效果、`lightUpEffect` 压暗提亮——逐像素颜色变换；
- 在自渲染图层**上方**叠加带模糊/高阶视效的 UI 控件——交叠区域同样得 GPU 混合。

视频播放页是最典型的踩坑场景：视频区域上盖一个毛玻璃控制栏，整个视频图层失去直通资格，GPU 每帧为视频区域全量工作。官方建议的改法是调整布局让模糊控件不与自渲染图层交叠，或改用非像素级视效（半透明纯色蒙版也要谨慎——取决于交叠范围与合成策略[待验证：半透明纯色蒙版在各版本是否必然破坏直通]）。

高阶视效本身的代价也有官方数据：《高效使用模糊》用 Frame 工具实测，背景模糊取色用 ColorPicker 时单帧平均 5.650ms，用 AdaptiveColor.Average 参数时单帧平均 9.400ms——同样是模糊，参数选错单帧多花近 4ms。

## 在 Trace 与工具中确认合成路径

- **看 RS 是否在为自渲染图层工作**：Buffer 直通形态下，自渲染图层附近看不到 RSUniRender 对它的重绘区间、DisplayNode 不刷新；被拖回 GPU 合成时，RenderFrame 段明显变长，且每帧都长（视频内容一直在变）。
- **功耗侧验证**：修改前后各抓一份功耗数据（DevEco Profiler 功耗会话或官方低功耗测试方法），对照官方 25.5%/30.7% 的量级评估自己的场景收益。
- **排查清单**：视频/相机/XComponent 场景功耗高 → ① 自渲染图层上有没有模糊、透明度、灰阶效果；② 上方悬浮控件有没有高阶视效；③ 图层有没有圆角裁剪之外的复杂变换。逐项移除后复查。

## 实际工作中的常见误区

**"帧率正常 = 渲染没问题"。** 帧率只反映时间维度。GPU 每帧给 4K 视频做全屏合成，帧率可以照样 60fps，但功耗和发热是另一条曲线。低功耗设计章节要单独走一遍。

**"模糊是系统做的，应该很便宜"。** 模糊是最贵的视效之一，面积越大越贵，而且还是 HWC 失效的头号原因。官方沉浸光感等文档都把这类材质称为稀缺视觉资源，要求控制面积与层数。

**RenderDeadlineMissed 就往 GPU 负载上归因。** GPU 负载高是一个方向，但 RS 侧超时也可能是图层数过多、事务处理队列堆积。先看 render_service 泳道里是 ProcessCommand 长还是 RenderFrame 长，再下结论。

## 版本演进

- API 9–11：统一渲染（UniRender）成为默认路径，RSNode 树与 DirtyRegion 机制成形。
- API 12（5.0）：同层渲染推广（Web、Video、XComponent 与 ArkUI 控件同层放置），官方开始发布 HWC 低功耗设计、Buffer 功耗优化等专项文档。
- API 20+（6.0）：高阶视效（沉浸光感材质等）增加，官方同步给出按算力分档的降级策略；自渲染图层直通规则在更多组件类型上生效。

## 参考资料

- 华为开发者文档：《高效利用 HWC 的低功耗设计》（bpta-utilize-hwc-efficiently：去除模糊后总功耗 -25.5%、CPU -30.7%）
- 华为开发者文档：《自渲染图层与 HWC 分析》（bpta-hwc-self-rendering-layer-analysis：避免高阶视效与自渲染图层交叠）
- 华为开发者文档：《Buffer 功耗优化》（bpta-buffer-power-optimization：直通形态 Trace、opacity/blur/grayscale/lightUpEffect 导致 GPU 重绘）
- 华为开发者文档：《高效使用模糊》（ui-use-blur-efficiently：ColorPicker 5.650ms vs AdaptiveColor.Average 9.400ms）
- 华为开发者文档：《沉浸光感约束》（arkts-immersive-light-sense-constraints：视效作为稀缺资源、控制面积与层数）
- 华为开发者文档：《丢帧分析》（bpta-zhenlv：Render Service 职责与绘制三阶段）
- OpenHarmony 仓库 `openharmony/graphic_graphic_2d`（render_service 进程、RSNode、RSUniRender、DirtyRegion、HWC 对接）
