# 3.3 手势识别与系统手势导航

## 手势是事件的"模式识别"

Touch 事件是原始的：按下、移动、抬起，一串坐标点。应用真正想响应的是"点击""长按""拖动""双指捏合"这类**模式**。ArkUI 的手势机制就是内置的模式识别器：事件在响应链上流动时，链上注册的手势识别器各自累积事件，满足条件就宣告识别成功，触发回调。官方 NDK 文档的定义：手势是一系列基础输入事件不断上报积累、达成一定特点时被识别成的交互结果，它能屏蔽不同基础输入事件的差异（触摸、鼠标、手写笔都归一到同一套手势）。

内置识别器覆盖常见模式：TapGesture（点击）、LongPressGesture（长按）、PanGesture（拖动）、PinchGesture（捏合）、RotationGesture（旋转）、SwipeGesture（快滑）。单个识别器挂到组件上就能用：

```typescript
Column()
  .gesture(
    PanGesture({ distance: 5 })
      .onActionStart(() => { /* 手指拖动达到阈值 */ })
      .onActionUpdate((e) => { /* 持续拖动，e 有偏移量 */ })
      .onActionEnd(() => { /* 抬起 */ })
  )
```

## 识别器之间的竞争：手势组

一个组件上同时绑了点击和拖动手势时，谁先触发？规则由手势绑定方式和手势组（GestureGroup）的模式决定：

- **Exclusive（互斥）**：先达到触发条件的手势胜出，其余失败。官方示例：组件上绑 TapGesture + PanGesture({distance: 5}) 互斥组，点一下就响应点击，滑动达到阈值就响应滑动，二者只触发其一。
- **Parallel（并行）**：组内手势可同时识别成功，比如旋转和捏合一起做。
- **Sequence（顺序）**：按声明顺序依次识别，前一个成功后才开始识别后一个。

绑定方式也分三档，优先级不同：`gesture`（默认）、`priorityGesture`（优先于子组件内置手势）、`parallelGesture`（允许与高优先级手势并行）。官方手势冲突 FAQ 给了一个典型案例：Image 组件内置的长按动画和开发者自定义的 LongPressGesture 冲突，改用 `priorityGesture` 绑定就只响应自定义手势；两个都要就用 `parallelGesture`。

父子组件的手势也存在天然竞争：子组件绑了 PanGesture，在子组件区域滑动默认只触发子的 Pan；想让父组件也响应，要么改绑定方式，要么调 distance 参数让灵敏度符合预期（官方单手势文档的建议），要么用嵌套滚动接口做协调。

API 12 之后还有一组更精细的钩子：`onGestureRecognizerJudgeBegin`（手势拦截，拿到识别器、设置开闭状态）、`shouldBuiltInRecognizerParallelWith`（系统组件内置手势与其他手势并行）；API 26 起 `onGestureCollectIntercept` 可以指定某个识别器或触摸识别器是否透传到其他节点。嵌套滚动联动这类复杂交互，官方手势判定文档给的示例就是用这些接口控制内外 Scroll 的滚动分工。

## 系统手势导航与应用手势的冲突

还有一层竞争在应用之外：系统导航手势。全面屏手势导航下，屏幕底部上滑回桌面、侧边缘内滑返回，这些手势由系统侧优先识别。应用如果也在边缘放了自己的滑动手势（侧滑抽屉、边缘返回卡片），就会撞上系统手势。

处理思路分三类：

1. **避让**：交互设计上避开系统手势热区，这是最省事的方案；
2. **声明优先**：侧滑返回这类场景，应用可用手势拦截与判定接口在指定区域声明应用手势优先，系统手势让位[待验证：边缘手势避让的具体 API 名称与适用范围，各版本有变化]；
3. **沉浸式协商**：全屏沉浸场景（视频、游戏）下系统手势降级为"二次确认"（第一次滑动只露出提示条），由系统管控，应用通过窗口属性配合。

实际开发中最常见的冲突是 Tabs/Swiper 这类横向滑动容器贴边放置：用户在边缘想触发系统返回，却拉动了应用的横向翻页，或者反过来。官方 PanGesture 文档里那条建议——Tabs 组件滑动与 PanGesture 并存时把 distance 设为 1 让滑动更灵敏——就是在这类冲突里调手感的具体手段。

## 在 Trace 与日志中看手势

手势识别本身没有独立泳道，排查靠间接证据：

- **识别延迟**：从 Down 到手势 onActionStart 的时间差，对应识别器的判定窗口（长按的时间阈值、Pan 的距离阈值）。用户抱怨"反应慢"时先核对阈值，别急着查渲染。
- **识别竞争失败**：onActionCancel 被调用说明识别到一半被打败——常见是父容器的手势或系统手势抢走了事件流。沿响应链排查各层绑了什么。
- **事件流断点**：在 Trace 的 Events 阶段对照回调日志，事件流中途不再到达某个识别器，说明上游某层做了拦截或 stopPropagation。

## 常见问题与误区

**"手势不触发就是组件没写对"。** 更多时候是竞争失败。先查绑定方式（要不要 priorityGesture）、手势组模式、distance/时间阈值这三件事，再查代码。

**"系统手势可以被禁掉"。** 不行，只能协商避让。试图完全屏蔽返回手势的应用过不了审核，也破坏了用户的系统级肌肉记忆。

**手势回调里做重活。** onActionUpdate 是热路径（每报点触发一次），和 onTouch 的 Move 回调同样要求轻量。拖动手感差，先查这里有没有同步耗时。

## 版本演进

- API 9–11：基础手势识别器与 gesture/priorityGesture/parallelGesture 绑定方式齐备；
- API 12（5.0）：手势判定与拦截接口（onGestureRecognizerJudgeBegin 等）开放，嵌套滚动协调成为官方推荐模式；
- API 18+：组合手势 NDK 接口完善（单一手势 + 组合手势，顺序/并行/互斥）；API 26 起 onGestureCollectIntercept 提供更细的透传控制；智慧手势（SmartGesture）事件支持注册监听回调动态决策手势归属。

## 参考资料

- 华为开发者文档：《NDK 事件响应》（ndk-add-event-response：手势是事件积累识别成的交互结果，单一/组合手势）
- 华为开发者文档：《多层级手势事件》（arkts-gesture-events-multi-level-gesture：Exclusive 手势组示例）
- 华为开发者文档：《单一手势》（arkts-gesture-events-single-gesture：父子 PanGesture 竞争与 distance 调整）
- 华为开发者文档：《手势冲突 FAQ》（arkts-gesture-event-conflict-faq：Image 内置长按动画冲突与 priorityGesture）
- 华为开发者文档：《PanGesture》（ts-basic-gestures-pangesture：Tabs 场景 distance 设 1）
- 华为开发者文档：《手势判定与拦截》（arkts-gesture-events-gesture-judge：onGestureRecognizerJudgeBegin、shouldBuiltInRecognizerParallelWith）
- 华为开发者文档：《智慧手势事件》（arkts-common-events-smartgesture-event）、`ts-gesture-blocking-enhancement`（onGestureCollectIntercept，API 26）
- OpenHarmony 仓库 `openharmony/arkui_ace_engine`：`frameworks/core/components_ng/gestures/`（识别器与手势组实现）
