# 3.1 多模输入框架与事件分发

## 为什么是"多模"

一台鸿蒙设备可能同时接着触摸屏、键盘、鼠标、手写笔，2in1 设备还有触控板，车机有旋钮和手柄。如果每种输入设备各走各的协议、各报各的格式，应用和窗口系统就要为每种设备写一套适配。多模输入框架（Multimodal Input，MMI，OpenHarmony 仓库 `foundation/multimodalinput/input`）做的事情是：**把所有输入设备的事件统一成一套标准输入事件流**，按统一的规则分发给当前该接收它的窗口。

官方交互基础机制文档把这条链概括为一句话：硬件输入设备通过驱动、多模等模块，将事件上报至目标的 ArkUI 实例，ArkUI 在渲染管线中进行统一处理。

## 分发链：从内核到组件

一次触摸的完整旅程：

```
触控 IC → 内核 input 子系统（/dev/input/eventX）
→ 多模输入服务（读取、加工、坐标变换、注入判定）
→ 窗口管理（按焦点/坐标确定目标窗口）
→ 应用进程 ArkUI 实例（管线 Events 阶段）
→ 触摸测试建立响应链 → 分发到组件
```

逐段说明。

**内核到多模服务。** 输入设备在内核里表现为 input 设备节点，多模输入服务的事件读取线程从设备节点读出原始事件（触点坐标、压力、时间戳、按键码）。这一段完全在用户感知之外，但原始采样率（比如触屏 120Hz/240Hz 报点）决定了后续所有环节的输入密度。

**多模服务加工。** 原始事件在这里被标准化：屏幕旋转后的坐标变换、多屏设备的屏 ID 映射、触摸/鼠标/手写笔源类型标记（官方手势设置文档里 SourceType 枚举就有 TOUCHPAD、JOYSTICK 等，触控板单指输入被视为鼠标输入）。事件注入也发生在这一层——自动化测试注入的事件在这里汇入同一条流，但会被打上"注入事件"标记。

**窗口管理定目标。** 指向性事件（有坐标的触摸、鼠标）按坐标找命中窗口；非指向性事件（按键）发给当前获焦窗口。官方键盘交互文档描述了按键事件的派发次序：窗口拿到按键后最多尝试三次分发——先给输入法（文字联想场景），再走组件焦点链，应用可用 `onKeyPreIme` 提前感知；电源键等系统按键则不传给应用。

**ArkUI 实例内分发。** 事件到达应用进程后，并不是立刻回调 `onTouch`，而是排队等待下一个 Vsync，在渲染管线的 Events 阶段（2.2 节七阶段的第二段，`FlushTouchEvents()`）统一分发。触摸事件经触摸测试（hit test）建立事件响应链后按链分发——这是 3.2 节的主题。

[图：输入事件分发链。左起：触控 IC/键盘/鼠标三个图标汇入"内核 input 子系统"；箭头到"多模输入服务"框（内部：事件读取线程、坐标变换、注入通道汇入点）；箭头到"窗口管理"（按坐标/焦点选窗口）；箭头进入应用进程框："ArkUI 渲染管线（Events 阶段）"→"触摸测试"→"组件响应链"。顶部标注时间维度：事件排队至下一个 Vsync 才被分发。]

## 应用侧接收入口

组件上最基础的接收入口是 `onTouch`（API 7 起就有，Stage 模型沿用）：

```typescript
Column()
  .onTouch((event: TouchEvent) => {
    if (event.type === TouchType.Down) {
      // 按下：event.touches 里有触点坐标、时间戳
    } else if (event.type === TouchType.Move) {
      // 移动：event.changedTouches 是变化的触点
      // event.getHistoricalPoints() 可取历史点，做平滑轨迹用
    }
  })
```

触摸事件对象里有几样东西对性能分析有用：触点位置、触点变化、历史点（一帧内多次采样的合并上报），以及时间戳——拿它和当前时间对比，可以估算事件在队列里排了多久。

## 事件注入与自动化验证

多模框架提供"以编程方式构造按键、鼠标、触屏、轴事件送入系统输入通道"的注入能力（官方 Input Kit 术语表）。约束明确：要按事件状态约束注入（按键先按下后抬起、轴事件先开始后更新再结束、同一触点不能重复按下），且受权限管控——系统侧要 `ohos.permission.INJECT_INPUT_EVENT`，开放侧要 `ohos.permission.CONTROL_DEVICE` 或用户授权。注入事件带标记，应用可识别。

日常验证用的是 uitest 框架或命令行：

```bash
hdc shell "uitest uiInput click 500 800"        # 模拟点击
hdc shell "uitest uiInput fling 500 1500 500 300 800"  # 模拟抛滑
hdc shell "uitest uiInput keyEvent Home"        # 模拟按键（API 18 起支持 inputText 等）
```

性能复现脚本（比如固定速度抛滑列表抓丢帧）就靠它保证每次输入一致。

## 实际排查：事件没到应用，查哪一层

"点了没反应"类问题的标准顺序：

1. **先确认设备层面有事件**：`hdc shell "hilog | grep -i input"` 看多模服务是否收到并上报；也可以用 hidumper 查多模服务状态[待验证：多模输入服务的 SA dump 命令与输出项]。
2. **确认窗口目标是否正确**：触摸点下有没有另一个透明窗口拦着（悬浮窗、未销毁的弹窗）。`hidumper -s WindowManagerService -a '-a'` 看窗口层级。
3. **确认应用是否收到**：在组件 onTouch 里打日志，或在 Trace 里看该 Vsync 周期 Events 阶段有没有分发记录。应用没收到而窗口正确，多半是主线程阻塞导致事件排队——这时用户感知是"点了过一会儿才反应"，对应第 8 章的响应时延问题。

## 版本演进

- API 9–11：多模输入框架随标准系统成熟，触控板、手写笔源类型补齐；
- API 12（5.0）：单框架后输入链路统一收进 ArkUI 管线 Events 阶段，注入事件标记与权限模型规范化；
- API 18+：uitest 注入命令持续增强（inputText 直输文本等）；API 26 起 ArkUI 增加 `onGestureCollectIntercept` 等更细的手势/触摸透传干预能力。

## 参考资料

- 华为开发者文档：《交互基础机制说明》（arkts-interaction-basic-principles：驱动→多模→ArkUI 实例、渲染管线统一处理）
- 华为开发者文档：《键盘交互开发指南》（arkts-interaction-development-guide-keyboard：三次分发、onKeyPreIme、系统按键）
- 华为开发者文档：《触摸事件 onTouch》（ts-universal-events-touch：触点、历史点、API 7 起）
- 华为开发者文档：《Input Kit 术语表》（input-kit-glossary：事件注入能力与权限、注入事件标记）
- 华为开发者文档：《uitest 使用指南》（uitest-guidelines：uiInput click/fling/keyEvent/inputText）
- OpenHarmony 仓库 `openharmony/multimodalinput_input`（多模输入服务）
