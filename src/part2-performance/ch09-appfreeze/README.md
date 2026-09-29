# 第 9 章：应用无响应（AppFreeze）

第 7、8 章讨论的卡顿和时延，是"慢"；本章讨论的 AppFreeze，是"停"。用户点击没有任何反应、界面定格不动，几秒后应用被系统杀掉——这是应用质量事故中最严重的一类，华为应用市场的上架质量指标里，应用无响应率与崩溃率并列。

AppFreeze 对应 Android 开发者的老朋友 ANR（Application Not Responding），思想一脉相承：系统给应用主线程设定响应时限，超时就判定故障、抓取现场、终止进程。但两者在检测维度和日志形态上有实质差异：AppFreeze 从主线程任务队列、用户输入、UIAbility 生命周期三个维度检测；日志里除了堆栈，还有 EventHandler 任务队列 dump、BinderCatcher 对端通信信息、整机 CPU 与内存快照，信息量远大于 Android 的 traces.txt。从 Android 迁移过来的读者可以带着 ANR 的经验来，但要重新熟悉这套日志的结构。

本章两节分工：

**9.1 AppFreeze 判定机制与故障类型。** 讲清楚三类主故障是怎么被判出来的：watchdog 线程定期向主线程插入判活任务，3 秒未执行告警（THREAD_BLOCK_3S）、6 秒未执行判死（THREAD_BLOCK_6S）；用户输入事件超过阈值无响应回执（APP_INPUT_BLOCK）；UIAbility 生命周期切换超时（LIFECYCLE_TIMEOUT，Load 10s、Foreground 5s）。还会给出官方的根因分类体系——4 类二级根因、17 类三级根因，以及与 Android ANR 的逐项对照。

**9.2 AppFreeze 日志分析与定位。** 拿到一份 appfreeze 日志后怎么读。日志固定分几个模块：头部信息（Reason、页面切换轨迹）、MSG 与 EventHandler 队列 dump、故障进程堆栈、BinderCatcher/PeerBinder 对端信息、CPU 使用率、内存信息。这一节逐段拆解每个模块"看什么、说明什么"，并给出三种典型形态的定位路径：主线程跑耗时任务（看堆栈栈顶）、等锁（对比 3s 与 6s 两份堆栈）、IPC 等待（看 BinderCatcher 的 wait 时长与对端堆栈）。API 21 起的增强日志（采样栈）解决了"瞬时栈抓不到真凶"的老大难，也会一并介绍。

一个前置提醒：通过 DevEco Studio 的 Debug 按钮安装并启动应用时，系统会自动关闭当前工程的超时检测机制，避免调试被打断（出处：《AppFreeze（应用冻屏）检测》）。所以调试期复现 AppFreeze，要用 Run 而不是 Debug，或者参考 9.1 节末尾的 aa attach/aa detach 命令控制检测开关。

读完本章，面对一份 AppFreeze 日志，我们应该能在五分钟内回答：哪类故障、主线程当时在干什么、是应用的问题还是系统/对端的问题、下一步改哪里。
