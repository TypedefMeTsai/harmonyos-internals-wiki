# 20.1 布局、状态与组件复用

## 一个先有共识：布局性能的变量是节点数量

官方《布局优化指导》做过一组对照实验：同样数量的 Row 容器，一组深层嵌套（10/100/500/1000 层）、一组平铺（同样数量的 Row 并排），测首帧绘制、Measure、Layout 耗时。结果两组的劣化趋势几乎一致——1000 个节点时嵌套组首帧 32ms、平铺组 24.3ms，差距存在但两者都已远超 16ms 帧预算。结论是：**嵌套深度本身不是决定性因素，参与布局的节点总数才是**。优化布局的第一优先级不是"把层数拍扁"这个动作本身，而是把节点数减下来。

这决定了后面所有手段的排序：先删冗余节点，再用扁平化布局减中间节点，再用布局边界限制更新范围，最后用懒加载和复用让节点"按需存在"。

## 手段一：移除冗余节点

最常见的冗余是同向容器套同向容器：Row 里只放了一个 Row，或者 Column 套 Column 只为加个 padding。这类节点不参与任何布局决策，纯粹增加 Measure/Layout 的递归开销。Code Linter 有现成规则 `@performance/hp-arkui-remove-container-without-property` 扫描"没有任何属性设置的容器组件"。

在静态页面上删一层容器收益有限，但在列表项里删一层意义很大——列表项的节点数会乘以缓存池大小和预创建数量。养成习惯：写完列表项布局后数一数节点数，Item 布局的节点预算是整个列表性能预算的除数。

## 手段二：扁平化布局替换

当布局没有冗余但嵌套依然很深时，换布局类型。官方示例：一个 4 层嵌套、15 节点的线性布局改用 RelativeContainer 后是 2 层、10 节点。可用的扁平化工具：

- **RelativeContainer**：用相对关系描述位置，替代多层 Row/Column 嵌套。
- **绝对定位（position/markAnchor）**：适合元素位置彼此独立的场景。
- **Grid**：二维布局直接描述，避免 Row 套 Column 的矩阵式嵌套。

但布局组件本身有性能差。官方在"5 层嵌套、20 个 Text 节点"的相同条件下对比首帧绘制：Column/Row 7.13ms、Stack 7.34ms、RelativeContainer 9.13ms、Flex 11.71ms、Grid/GridItem 12.62ms。Flex 的二次布局使其 Measure 耗时是 Column 的近 3 倍。选型原则：

- 同层数同节点数能实现相同效果时，选 Column/Row/Stack，不用 Flex 凑单行布局。
- 能用高级容器大幅减少节点数时（RelativeContainer 拍扁深嵌套），容器本身的差价被节点减少的收益覆盖，值得换。
- Flex 留给真正需要折行的场景，Grid 留给真正的二维网格。

## 手段三：用布局边界限制更新范围

ArkUI 的标脏机制：布局属性（width/height/padding/margin）变化会标记"布局脏"并向上找布局边界做子树更新；非布局属性（color/backgroundColor/opacity）只影响自身。**设置了固定宽高的组件就是布局边界**——它内部的变化不会触发外部的重新测算。

官方实测很有说服力：一个包含 400 条文本的 Row，外层容器宽度变化时——Row 固定宽高（300×400）的重绘耗时 2.0ms，未设宽高的 38.45ms，设百分比宽高的 42.62ms，差距约 20 倍。固定宽高的组件在重绘时直接使用初次绘制保留的节点大小数据，跳过整个子树的 Measure。

**怎么做**：对尺寸不随内容变化的容器（卡片、列表项骨架、固定工具栏），在 UI 描述时给定数值宽高。内容自适应的组件（文本长度不定）保持自适应，但可以在更外层套一个固定尺寸的容器当边界。

**怎么验证**：Frame 会话中触发尺寸变化，看 FlushLayoutTask 里参与布局的节点数量是否收敛到边界内；对比重绘帧的 Measure/Layout 耗时。

## 手段四：if 与 visibility 的显隐选型

两者机制不同：if 控制组件是否**创建**，visibility 控制组件是否参与**布局渲染**。官方用 100 个 Image 组件的 Column 做显隐对照：

- 初次加载：if 为 false 时组件创建 3.83ms（不创建），visibility=None 时 13.26ms（照样创建）。
- 切换显隐：if 切到 true 时 Measure 3.10ms（要新建组件再测算），visibility 切换只要 0.19ms。

选型规则：频繁切换显隐（Tab 内容、折叠面板）用 visibility，省掉反复创建；创建昂贵且不常出现的面板（设置页深层入口）用 if，初次不创建省内存省首帧。这个结论和第 19 章"首页用 if 隐藏非首帧组件"是同一个机制的两面。

## 手段五：@Builder 与自定义组件的取舍

@Builder 是编译期展开的构建函数，不产生独立的组件节点和状态观测单元；@Component 是完整组件，有自己的生命周期、状态订阅和复用资格。性能上的差别：

- **更新范围**：@Builder 里的状态变化会把更新回溯到宿主组件，触发宿主 build 重跑；自定义组件的 @State 变化只重建自己。频繁更新的局部内容（秒表、进度条）应该拆成独立组件，让更新收敛在小组件内。
- **创建成本**：自定义组件有实例化开销，纯展示、无状态、量大的小片段用 @Builder 更省。

一句话：更新频繁的拆组件，静态重复片段用 @Builder，不要反过来。

## 手段六：状态精准拆分

状态管理是更新范围的另一个决定因素。反面模式：把页面所有数据塞进一个大对象用 @State 持有，任何一个字段变化都触发整棵组件树的脏检查与重建。正确做法：

- 状态按"消费它的 UI 范围"拆分，哪个小组件用哪个字段，字段就放到哪个小组件的 @State/@Prop 里。
- 大对象用 @Observed/@ObjectLink 做属性级观测，让只有被修改的属性关联的组件刷新。
- 列表数据用 LazyForEach 的数据源通知机制（onDataChange 等）精确到项，避免 notifyDataReloaded 全量重刷。
- 避免在 build 里做昂贵计算和对象深拷贝——build 每次刷新都执行，里面每分配一个大对象都是 GC 压力和耗时。

**怎么验证**：DevEco Profiler 的 Frame 场景支持状态变量维度的分析（看一次状态变化触发了多少组件刷新）；也可以在可疑组件的 build 里加 HiTraceMeter 打点，数一次交互触发了几次 build。

## 手段七：LazyForEach + cachedCount + @Reusable

长列表的三件套，官方数据（《布局优化指导》引用长列表最佳实践）：

**懒加载**：10000 条数据时，ForEach 的列表挂载时间 3291ms、独占内存 560.1MB、丢帧率 58.2%；LazyForEach 对应 97ms、82.9MB、6.6%。100 条以内两者差距不大，用 ForEach 代码更简单；超过两屏必须用 LazyForEach（或 API 19+ 的 Repeat 数据精准懒加载；状态管理 V2 里官方推荐 Repeat + virtualScroll 替代 LazyForEach）。

**cachedCount**：可视区域外预创建/预布局的缓存项数量。值太小，快速滑动时新项来不及创建，出现白块和 BuildLazyItem 耗时尖峰；值太大，首帧创建量上升、内存上涨。从默认值出发按滑动速度调，用 Frame 会话看快速滑动时的 BuildLazyItem 分布。

**组件复用（@Reusable）**：滑动中组件不再创建-析构，滑出缓存区的组件进复用池，新进入的从池中取出更新数据。官方对比：开启复用后丢帧率 3.7% → 0%，BuildLazyItem 耗时 10.277ms → 0.749ms，复用路径的 BuildRecycle 仅 0.221ms。复用的注意事项：

- 复用组件的 aboutToReuse 回调里重置状态，否则旧数据会闪现在新项上。
- 列表项布局差异大时按类型拆分多个复用池（不同 itemType），避免复用到布局不符的节点。
- 状态管理 V2 用 @ReusableV2 与 Repeat 配套。

## 手段八：两个结构性反模式

**Scroll 嵌套 List 不给 List 宽高**：Scroll 的可滚动方向尺寸为无穷大，List 不设高度时高度也是无穷大，LazyForEach 的视口裁剪失效，所有 ListItem 全部参与布局。官方实测：100 条数据的 LazyForEach，不设宽高时布局任务 100 个、布局时间 32.43ms；设了高度后布局任务 12 个、6.08ms。同理，LazyForEach 在无边界约束的容器里会在单个布局帧内瞬时实例化数千个组件，导致懒加载机制瘫痪、UI 线程阻塞甚至 OOM。规则只有一条：**任何懒加载容器必须有明确的高度约束**（固定值、百分比或 layoutWeight）。

**首页堆全量组件**：首帧所有组件（除 if 不成立分支和 LazyForEach 不可视区）都走完整的 Build/Measure/Layout/Render，首页节点数直接决定首帧耗时。首屏只放可见内容，折叠内容用 if 或 LazyForEach 延迟。

## 手段九：NodeContainer 与自绘节点

对于框架组件表达不了或表达成本太高的场景（复杂图形、游戏 HUD、自绘图表），可以用 BuilderNode + NodeContainer 脱离组件树管理自己的 FrameNode 子树。这是逃生舱不是常规武器：它绕过声明式状态管理，更新全手动，只在组件树开销确实成为瓶颈且有明确证据时使用。

## 验证方法汇总

- 节点数与布局耗时：Frame 会话的 FlushLayoutTask/FlushRenderTask 看布局任务数量；DevEco 的 ArkUI Inspector 看组件树结构。
- 滑动丢帧：Frame 会话的 Frame 泳道看 AppDeadlineMissed 红帧，丢帧帧点击查看 BuildLazyItem/FlushLayoutTask 耗时。
- 显隐/状态更新范围：状态变量分析或 HiTraceMeter 自打点计数。
- 复用效果：对比 BuildLazyItem 与 BuildRecycle 的出现频率和耗时。

## 参考资料

- 官方文档：《布局优化指导》（arkts-layout-optimization-guidance）
- 官方文档：《状态管理最佳实践》（bpta-status-management）
- 官方文档：《LazyForEach 性能优化》（bpta-lazyforeach-optimization）
- 官方文档：《长列表加载丢帧优化》（bpta-best-practices-long-list）
- 官方文档：《瀑布流性能优化》（bpta-waterflow-performance-optimization）
- 官方文档：List 组件参考（cachedCount）、LazyForEach/Repeat 渲染控制参考
- 官方文档：《组件复用 V1/V2 迁移》（arkts-v1-v2-migration-reusable）
