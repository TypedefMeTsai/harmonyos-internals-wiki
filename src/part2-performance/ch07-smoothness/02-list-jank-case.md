# 7.2 滑动丢帧分析实战：长列表案例

本节完整复盘一个长列表滑动丢帧的分析过程。案例来自官方最佳实践《帧率问题分析》中以"HMOS世界"应用首页列表为对象的实战，文中所有数据均出自该文档与《ArkUI 布局优化指导》。长列表是丢帧问题最高发的场景，这个案例覆盖的三个根因——组件未复用、数据深拷贝、布局嵌套过深——几乎也是所有列表类卡顿的标准答案。

## 问题现象描述

"HMOS世界"应用（HarmonyOS 官方示例工程）首页是一个文章列表，为了演示效果，列表初始加载了 1000 条数据。设备为 120Hz 刷新率的手机。

现象：滑动列表时，随着滑动距离增加逐渐出现卡顿，列表项越多越卡。不是滑不动，而是画面不时顿一下，跟手感差。

复现条件很简单：打开首页，持续上下滑动即可。这类"越滑越卡"的现象本身就携带信息——它指向随数据量或滑动次数累积的开销，比如组件反复创建、内存压力上升导致 GC 频繁，而不是一次性的初始化耗时。

## 分析思路

长列表卡顿，第一嫌疑人是列表项的创建成本。ArkUI 中 List 配合 LazyForEach 只创建可视区域及前后少量缓存项，理论上滑动时每一帧只需要新建一两个列表项。但如果每个列表项的创建本身很贵，或者创建逻辑被不必要地放大，120Hz 下 8.3ms 的帧预算很容易被突破。

所以分析主线定为：先确认丢帧率和丢帧侧（App 还是 RS），再定位帧内耗时最长的阶段，最后落到具体的组件和函数。先体检拿线索，再录 Trace 证实，两步走。

## 抓取与定位

**第一步：AppAnalyzer 体检。** 在 DevEco Studio 中启动 AppAnalyzer，选择场景化体检中的"手动性能页面滑动体检"。工具自动编译、安装、运行工程，保持手机解锁；提示"体检中，请操作手机"时，在首页列表上滑动若干次，然后停止。体检报告很快给出三条线索：UI 线程应用自身方法耗时长、组件未有效复用、部分图片纹理过大。

**第二步：录制 Frame 模板。** 体检报告给了方向，但需要 Trace 证实。创建 Frame 分析任务（录制前在泳道配置里手动勾选 ArkUI Component 泳道，这个泳道默认不开），录制期间重复滑动列表。

**第三步：确认丢帧规模。** 在时间轴上框选 2.5 秒的滑动区段，选中 Frame 主泳道，Statistics 栏显示：该时间段内丢失 16 帧，丢帧率 7%。丢帧主要出现在 App Frame 子泳道——定界为 AppDeadlineMissed，应用侧问题。

[图：Frame 泳道框选 2.5 秒区段后的 Statistics 统计视图，显示丢帧 16 帧、丢帧率 7%，App Frame 子泳道散布红色帧。]

## 逐步分析

**看到了什么**：放大选中超时帧（220# 帧）查看详情，期望处理时间 8.3ms（120Hz），实际处理时间 8.9ms。

**说明了什么**：只超了 0.6ms 就丢帧，说明每一帧的预算几乎没有余量。这种"每帧都贴着预算线跑"的状态，通常意味着帧内有一个稳定存在的固定开销，而不是偶发的尖刺。

**下一步看哪里**：跳转到该帧的应用侧 Trace 详情，看帧内各阶段的耗时分布。

**看到了什么**：每个卡顿帧下都有 `Builder:BuildLazyItem` 打点，且耗时较长；ArkUI Component 泳道上，自定义组件 ArticleView 的绘制频率高且单次耗时明显。

**说明了什么**：滑动过程中列表项在逐帧新建，`BuildLazyItem` 是 LazyForEach 创建新列表项的入口，它的耗时就是列表项创建成本。嫌疑收窄到"列表项为什么这么贵"。

**下一步看哪里**：切到 ArkTS Callstack 泳道，看 220# 帧的热点函数。

**看到了什么**：热点函数中 `initialRenderView` 占比 52.7%，`__lazyForEachItemGenFunction` 占比 22.9%。展开 `initialRenderView`，主要耗时在列表项 ListItem 的子组件 ArticleCardView 的创建上；再展开调用链，发现子组件中使用了 `@Prop` 装饰的变量。

**说明了什么**：链条完整了。`@Prop` 装饰的变量会对父组件传入的状态值做深拷贝，装饰复杂对象或类实例数组时，每次创建组件都要付一份深拷贝的时间和内存。列表项组件没有开复用，滑动中每进入一个新区间的项就完整走一遍"创建组件 → 深拷贝 @Prop 数据 → 构建子树"的流程。1000 条数据的列表越滑，创建次数越多，卡顿越明显。

## 根因与结论

三个叠加的根因：

1. **组件未复用**：列表项组件 ArticleCardView 是普通 `@Component`，滑出屏幕即销毁，滑入就新建，`BuildLazyItem` 每帧都要付完整的创建成本。
2. **`@Prop` 深拷贝**：列表项内部用 `@Prop` 接收复杂数据，每次创建都深拷贝一遍数据，进一步放大单项创建成本。
3. （次级因素）体检报告同时指出了部分图片纹理过大，源图尺寸超过目标尺寸，增加解码和上传开销。

## 修复/优化方案

两处改动（出处：《帧率问题分析》）：

**一，开启组件复用。** 给列表项组件加 `@Reusable`，并在 ListItem 上设置 `reuseId`。可复用组件从组件树上移除时不销毁，进入回收缓存区；后续需要新节点时从缓存区取出，通过 `aboutToReuse` 回调刷新数据，跳过整个创建流程：

```typescript
List() {
  LazyForEach(this.dataSource, (item: LearningResource) => {
    ListItem() {
      ArticleCardView()
        .reuseId('article')
    }
  }, (item: LearningResource) => item.id.toString())
}

@Reusable
@Component
export struct ArticleCardView {
  aboutToReuse(params: Record<string, Object>): void {
    // 用新数据刷新组件状态
  }
  // ...
}
```

**二，用 `@Builder` 替代小组件，消灭 `@Prop`。** 列表项内部的按钮组等简单结构改用 `@Builder` 函数构建。`@Builder` 不创建新的自定义组件实例，自然也就不需要 `@Prop` 传参，深拷贝开销随之消失：

```typescript
@Builder
function ActionButtonBuilder() {
  // 直接引用外层数据，无需 @Prop 传参
}
```

配套检查项：确认 List 设置了合理的 `cachedCount`（缓存可视区前后各 N 项，默认按屏幕容量计算，增大它用内存换创建次数，但不要设得过大，否则首屏创建项过多反而拖慢加载）；确认列表项图片做了尺寸适配（沙箱图片可设 `autoResize`，网络图片换 WebP，源图不超过目标尺寸 10%）。

**验证效果**：修复后重新录制 Trace 并体检。官方《ArkUI 布局优化指导》给出的长列表案例对比数据：组件复用前丢帧率 3.7%，复用后 0%；`BuildLazyItem` 耗时复用前 10.277ms，复用后 0.749ms——复用路径下创建变成 `BuildRecycle`，耗时降了一个数量级。官方《帧率问题分析》中自定义动画案例也印证了"换用系统机制"的收益：自绘帧动画改为属性动画后，动画帧率从 63fps 提升到 116.9fps。

## 附带案例：布局嵌套过深

同一篇官方文档还给了另一个有教学价值的案例。列表加载 2000 条数据，列表项 ChildComponent 的布局嵌套了 20 层 Stack。Frame Profiler 录制后发现卡顿帧的 `FlushLayoutTask` 耗时过长——这个阶段对所有标脏组件做测量和布局，展开后 Measure 方法耗时占大头。用 ArkUI Inspector 查看真机组件树，可以直观看到 Item 的嵌套深度。

修复方式是把 20 层 Stack 拍平成一层（需要复杂对准时改用 RelativeContainer，一层布局表达相对位置关系），再次录制，丢帧消失。

这个案例的教训：Measure/Layout 的成本随嵌套深度近似线性增长，20 层嵌套意味着每个列表项的布局算法要递归走 20 层。120Hz 下这种结构性开销没有任何运行时手段能救，只能在布局设计上避免。

## 举一反三

这个案例的方法可以原样搬到以下场景：

**任何"越用越卡"的列表**（消息流、商品流、评论区）：框选区段看丢帧率 → 定界 App 侧 → 找 `BuildLazyItem` → ArkUI Component 看哪个组件贵 → ArkTS Callstack 找热点函数。`aboutToBeDeleted` 调用次数过多是另一个信号：destroy 只发生在 idle 时，如果它频繁出现，说明组件在反复销毁重建（没开复用），或者主线程阻塞导致销毁任务积压。

**Grid、WaterFlow、Swiper**：同样支持 LazyForEach 和 cachedCount，同样适用组件复用，分析路径完全一致。

**非列表的滑动卡顿**（如 Scroll 内嵌复杂内容）：没有 `BuildLazyItem` 可看，改看 `FlushLayoutTask` 和 `FlushRenderTask` 的耗时分布，怀疑嵌套深度和标脏范围。

**体检报告说"组件未有效复用"但没给细节**：工具通过实时检测组件的创建行为给出提示，结合组件复用问题诊断分析文档逐项核对——常见的复用失效原因包括 reuseId 冲突、`aboutToReuse` 里漏更新状态、组件内持有不可复用的重型资源。

最后记住一个优先级：长列表性能问题，先查懒加载（是不是误用了 ForEach 一次性加载全部），再查复用，再查单项成本（深拷贝、嵌套、图片），最后才轮到主线程其他耗时。按这个顺序，多数列表卡顿在前两步就能定位。

## 参考资料

- 《帧率问题分析》（bpta-zhenlv）
- 《ArkUI 布局优化指导》（arkts-layout-optimization-guidance）
- 《优化长列表加载慢丢帧问题》（bpta-best-practices-long-list）
- 《组件复用》（bpta-component-reuse）
- 《组件复用问题诊断分析》（bpta-component-reuse-issue-diagnosis-and-analysis）
- 《主线程耗时操作优化指导》（bpta-time-optimization-of-the-main-thread）
- 《LazyForEach：数据懒加载》（arkts-rendering-control-lazyforeach）
