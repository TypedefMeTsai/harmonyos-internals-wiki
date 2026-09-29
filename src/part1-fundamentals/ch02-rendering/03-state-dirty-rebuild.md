# 2.3 状态管理、脏区刷新与组件重建

## 一个变量改动之后发生了什么

看一行代码：`this.count++`，其中 `count` 是 `@State` 装饰的变量，页面里有三个 `Text` 显示了它，旁边还有一百个与它无关的组件。问题：下一帧系统会重建多少个组件？

这个问题是 ArkUI 性能的核心问题。官方布局优化文档描述了界面更新阶段的过程：触发列表滑动、显示/隐藏切换、元素属性变化时，UI 线程先将脏节点进行 Build，Build 过程按组件 id 依次更新组件设置的属性，属性发生改变则进行组件标脏。V1 和 V2 两代状态管理框架，回答这个问题的粒度不同——V1 是组件级，V2 做到了属性级。理解这个差异，才能解释为什么同样的页面，V2 下更新耗时会明显低。

## V1：依赖收集与组件级标脏

状态管理 V1 的机制分两步。

**依赖收集发生在 build 期间。** 自定义组件执行 `build()` 时，每读到一个被 `@State`/`@Prop`/`@Link`/`@Observed` 装饰的变量，框架就记录一条依赖：这个组件的更新函数订阅了这个变量。收集是动态的——下次 build 时重新收集，所以条件分支里读到的变量只有分支成立时才建立依赖。

**变量被赋值时标脏。** 赋值走装饰器拦截，框架查到所有订阅者，把这些自定义组件标记为脏（dirty），加入脏节点集合。注意粒度：**标脏的单位是整个自定义组件**。哪怕组件里只有一行 Text 用了这个变量，下一帧也会重新执行该组件的 build 函数，再对比属性变化决定哪些 FrameNode 需要进 Measure/Layout。

标脏不是立即重建。所有标脏动作只是把节点挂进脏集合，**批量合并到下一个 Vsync 的 UpdateUI 阶段统一处理**——这就是 2.2 节七阶段里 UpdateUI（`FlushBuild()`）的内容。一个事件回调里改十个状态变量，只会触发一轮重建，这是批量合并的收益。

`@Observed` 类有一个著名限制：只能观测对象第一层属性的变化（以及数组整体变化），嵌套对象内部属性改了不会触发刷新。这正是 V2 要解决的问题之一。

## V2：@ObservedV2/@Trace 与属性级更新

状态管理 V2（API 12 起）换了观测模型。核心装饰器：`@ObservedV2` 装饰类，`@Trace` 装饰类中需要被观测的属性；组件侧用 `@Local`、`@Param`、`@Event` 等。官方文档对这组能力的描述是：提供对嵌套类对象属性变化直接观测的能力，无论嵌套多少层都能观测。

机制上的两点不同：

**观测粒度到属性。** `@Trace` 属性被读取时，记录依赖的是"哪个 UI 绑定读了这个属性"；属性被写时，只有绑定了**该属性**的组件被标脏。官方术语表称之为"属性级更新"：属性变化时仅刷新该属性绑定的组件，避免未变化属性关联组件的连带刷新。前面例子里那种"改一个字段、整个卡片重建"的连带刷新，在 V2 下可以消掉。

**标脏时机异步化。** 官方 V1/V2 更新差异文档指出：相比 V1，V2 在状态变量变化时会**异步标脏**组件。表现上的一个差异是 `@Monitor` 回调要等当前事件逻辑（如 onClick）执行完才触发——赋值、继续执行、最后统一处理。写代码时不要假设赋值后下一行就能读到 UI 已刷新。

```typescript
@ObservedV2
class TaskInfo {
  @Trace title: string = '';     // 改 title 只刷新绑定 title 的组件
  @Trace done: boolean = false;
  note: string = '';             // 未 @Trace，改了不触发刷新
}

@ComponentV2
struct TaskCard {
  @Param task: TaskInfo = new TaskInfo();
  build() {
    Column() {
      Text(this.task.title)      // 只依赖 title
      if (this.task.done) {      // 只依赖 done
        Image($r('app.media.check'))
      }
    }
  }
}
```

上面这个组件里，`task.title` 变化不会导致 done 分支重新求值，反之亦然。嵌套多层的对象图（官方混用示例里三层嵌套的 MessageInfo）也能逐层精确观测。

## 组件重建与组件复用

标脏刷新解决"更新时少做事"，还有一类开销在**创建销毁**：LazyForEach 滑动时，新进入屏幕的列表项要完整走 Build→Measure→Layout（Trace 点 BuildItem / BuildLazyItem），划出屏幕的销毁。列表项组件树越复杂，这段耗时越大。

`@Reusable`（API 10 起）让组件可复用：划出屏幕的列表项不销毁，进入复用池；新项需要创建时，从池里取一个同类型实例，调 `aboutToReuse(params)` 喂新数据，跳过整个构造流程。Trace 上表现为 BuildItem 变成耗时极短的 BuildRecycle。官方布局优化文档给了长列表场景的对比数据：**开启组件复用后丢帧率从 3.7% 降到 0%，BuildLazyItem 耗时从 10.277ms 降到 0.749ms**。

```typescript
@Reusable
@Component
struct GoodsCard {
  @Prop item: string = '';
  aboutToReuse(params: ESObject) {   // 复用时被调，更新数据
    this.item = params.item;
  }
  aboutToRecycle(): void {           // 进入复用池时被调，可做清理
  }
  build() { /* 列表项 UI */ }
}
```

复用失效有个经典场景值得记住：官方复用问题诊断文档描述，匀速滑动时"一帧划出几个就复用几个"形成动态平衡；突然加速滑动（一帧划出两个），池中缓存不够，就会复用一个再新建一个——Trace 上看到 BuildRecycle 减少、BuildItem 重现，伴随内存上涨。看到这种形态，说明复用机制在，但供给跟不上消耗，可考虑增大 cachedCount 或预构建（NodePool + onIdle 在空闲帧预建）。

## 在 Trace 中看状态刷新

- `JSView: ExecuteRerender`：脏组件 build 重执行。这段长，说明被重建的组件多或单个 build 太重——先查标脏范围是否过大（V1 粗粒度、滥用 @ObjectLink 大对象），再查 build 内是否有耗时计算。
- `H:Builder:BuildLazyItem` / BuildItem / BuildRecycle：LazyForEach 列表项的创建与复用。滑动期间 BuildItem 密集出现而 BuildRecycle 稀少，按上文的供给模型分析。
- UpdateUI 阶段总长在 Frame 详情里可直接读，和七阶段预算对照。

## 实际优化清单

1. 大页面状态拆分：一个巨型 @State 对象改成多个小对象，V1 下减少整组件重建，V2 下配合 @Trace 做到字段级。
2. 嵌套数据模型迁 V2：多层嵌套且局部更新的场景（IM 消息列表、表单），@ObservedV2/@Trace 收益最大。
3. 长列表必须 LazyForEach + @Reusable + 合理 cachedCount；用官方两组数字（丢帧率 3.7%→0%，10.277ms→0.749ms）做收益预期。
4. 避免在 build 里做计算和 new 大对象：build 每帧可能重跑，里面的开销乘以帧率。
5. 别在同一个事件里反复改同一状态指望中间态刷新——批量合并只留最终结果。

## 版本演进

- API 9：V1 状态管理（@State/@Prop/@Link/@Observed/@ObjectLink）随 Stage 模型发布。
- API 10：@Reusable 组件复用。
- API 12：状态管理 V2 首批（@ComponentV2/@ObservedV2/@Trace/@Local/@Param/@Monitor），官方开始提供 V1→V2 迁移指南。
- API 20+：@ReusableV2 与全局复用池（reusePool/poolAccepts）、@SyncMonitor（API 23）等补齐 V2 生态；Repeat 等新渲染控制语法要求搭配 V2。

## 参考资料

- 华为开发者文档：《@ObservedV2 与 @Trace》（arkts-new-observedv2-and-trace，API 12 起）
- 华为开发者文档：《状态管理术语表》（arkts-state-management-glossary：属性级更新、深度观察定义）
- 华为开发者文档：《V1/V2 状态变量更新差异》（arkts-v1-v2-update-difference：V2 异步标脏）
- 华为开发者文档：《布局优化指导》（arkts-layout-optimization-guidance：标脏与 Build 过程、组件复用前后丢帧率 3.7%→0%、BuildLazyItem 10.277ms→0.749ms）
- 华为开发者文档：《@Reusable 组件复用》（arkts-reusable）、《组件复用问题诊断》（arkts-diagnosis-component-reuse-issues：动态平衡与加速滑动场景）
- 华为开发者文档：《LazyForEach 数据懒加载》（arkts-rendering-control-lazyforeach）
- OpenHarmony 仓库 `openharmony/arkui_ace_engine`：`frameworks/bridge/declarative_frontend/state_mgmt/`（V1 状态管理实现）、`frameworks/core/components_ng/base/frame_node.cpp`（脏节点标记）
