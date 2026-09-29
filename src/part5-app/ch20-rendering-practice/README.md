# 第 20 章：渲染优化实战

## 本章导读

第 7 章讲过丢帧的故障模型：AppDeadlineMissed 和 RenderDeadlineMissed 分别对应应用侧超时和 Render Service 侧超时。第 2 章讲过一帧的生命周期：Vsync 到来，应用侧 Build/Measure/Layout/Render，事务提交给 RS，RS 合成送显。那些是"看懂 Trace 需要的知识"。这一章回答的是下一个问题：**Trace 里看到 Build 慢、Render 慢、HWC 没生效之后，代码应该怎么改**。

本章按问题发生的层分三节：

**20.1 布局、状态与组件复用**。应用侧超时的两大来源：一是参与布局的节点太多（嵌套深、容器冗余、该扁平的没扁平），二是状态变化引发的更新范围太大（大对象 @State 一变，整棵树重建）。这一节给出布局扁平化、布局边界、if 与 visibility 的选型、@Builder 与组件的取舍、状态精准拆分、LazyForEach + @Reusable + cachedCount 的完整打法，全部配官方实测数据。

**20.2 动画、图片与高阶视效**。动画卡在"每帧都在布局"还是"每帧只改图形属性"，图片卡在"解码分辨率"和"上传方式"，高阶视效（模糊、反色、提亮）卡在"离屏渲染"和"HWC 失效"。这一节讲属性动画与帧动画的选型、renderGroup 把动画挪到 RS 侧缓存的机制、ImageKnife 与解码尺寸匹配、以及模糊类视效与自渲染图层交叠导致 HWC 失效的机理。

**20.3 ArkWeb、视频与 XComponent 场景**。ArkWeb 页面有自己的渲染进程和自己的丢帧模式（JSView:ExecuteRerender、RosenWeb buffer），视频和 XComponent 是自渲染图层，与统一渲染管线的合成策略直接相关。这一节讲 Web 组件预启动/预渲染、Web 丢帧的 Trace 读法、视频图层与模糊控件交叠的 HWC 问题、同层渲染的取舍。

三节的共同方法：先想清楚这条优化改变了渲染管线里的哪一环，再决定用 Frame 会话的哪个泳道、哪个 Trace 点验证。改代码之前没有这条链路图，改完之后你就不知该看哪里。
