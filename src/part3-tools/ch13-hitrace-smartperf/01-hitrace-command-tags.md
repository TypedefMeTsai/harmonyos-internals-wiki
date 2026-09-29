# 13.1 HiTrace：bytrace/hitrace 命令与 tag 体系

## 这个工具解决什么问题

系统运行时在关键路径上埋了成千上万个打点：ArkUI 每次刷新、RS 每次合成、Binder 每次 IPC、调度器每次切换，都有对应的 trace 记录。这些打点平时写在内核的环形缓冲区里，不占什么成本；出问题时，我们需要一个工具把缓冲区里某段时间的内容倒出来——这就是 `hitrace` 命令。它通过 hdc shell 在设备上执行，支持采集系统打点和应用用 HiTraceMeter 接口打的点，输出文本或二进制两种格式（出处：《hitrace》）。

环境准备只有两步：配好 hdc（参考 hdc 环境准备文档），`hdc shell` 进入设备。所有命令在设备 shell 里执行。

## 三种采集模式

hitrace 有三种工作模式，按场景选用：

**模式一：定时采集（最常用）。** 指定时长，到点自动停止并输出：

```shell
# 采集 10 秒、缓冲区 204800KB、只采 app tag，输出到屏幕
hitrace -t 10 -b 204800 app

# 输出到文件（文本格式建议放 /data/local/tmp）
hitrace -t 10 -b 204800 app -o /data/local/tmp/test.ftrace

# 二进制格式（固定输出到 /data/log/hitrace，结尾打印文件路径）
hitrace -t 10 -b 204800 app --raw

# 多 tag + 压缩
hitrace -z -b 102400 -t 10 sched freq idle disk -o /data/local/tmp/test.ftrace
```

**模式二：快照模式（手动控制起止）。** 适合复现类问题：先开采集，手动操作复现，复现完立刻停止，抓到的就是故障现场：

```shell
hitrace --trace_begin -b 204800 app graphic   # 开始
# ……在设备上操作复现问题……
hitrace --trace_dump -o /data/local/tmp/test.ftrace  # 导出（不停止）
hitrace --trace_finish -o /data/local/tmp/test.ftrace # 停止并导出
hitrace --trace_finish_nodump                        # 停止不导出
```

快照模式下还有一组二进制命令：`--start_bgsrv` 开启、`--dump_bgsrv` 导出、`--stop_bgsrv` 停止。注意二进制快照模式**不支持指定 tag**，默认采集一组固定 tag（net、graphic、multimodalinput、ark、ace、window、app、ability、power、ffrt、sched、freq、binder 等约 30 个），对性能分析来说这组默认 tag 基本够用。

**模式三：录制模式（长时间落盘）。** 持续采集二进制 trace 并滚动写入文件，文件写满设定大小后自动开新文件，适合稳定性压测和长场景：

```shell
hitrace --trace_begin --record -b 204800 --file_size 102400 app graphic
# ……长时间运行……
hitrace --trace_finish --record
# 输出示例：/data/log/hitrace/record_trace_20250604170337@614463-183970330.sys（可能多个文件）
```

录制模式从 API 24 起支持 `-o /data/local/tmp` 指定输出目录和 `--total_size` 限制总大小。

关键参数汇总（出处：《hitrace》）：`-b N` 缓冲区 KB（默认 18432，最小 512）；`-t N` 时长秒（默认 5）；`--file_size` 单文件 KB（默认 102400）；`--overwrite` 缓冲区满后丢新数据（默认丢老数据——排查问题时默认行为更合理）；`--trace_clock` 时钟源（默认 boot，休眠也计时，一般用默认）。

## tag 体系

`hitrace -l` 列出全部 tag。70 多个 tag 分用户态和内核态两类，性能分析高频使用的如下（出处：《hitrace》tag 表）：

| tag | 内容 | 典型场景 |
| --- | --- | --- |
| app | 应用模块（HiTraceMeter 打点全在这个 tag） | 一切应用分析，必带 |
| ace | ArkUI 跨平台引擎 | UI 刷新、组件渲染 |
| graphic | 图形模块 | RS 侧渲染、合成 |
| window | 窗口管理器 | 窗口切换、转场 |
| ability | 能力管理器服务 | 生命周期、启动 |
| ohos | 系统通用标签 | 系统服务通用打点 |
| ffrt | FFRT 任务 | 任务调度、并发 |
| ark | Ark 模块 | 方舟运行时、GC |
| multimodalinput | 多模态输入 | 触摸事件、响应时延 |
| sched | CPU 调度（内核态） | 线程状态、唤醒链 |
| freq | CPU 频率（内核态） | 大小核、限频 |
| binder | Binder 通信（内核态） | IPC 阻塞 |
| disk / pagecache | 磁盘 I/O、页缓存（内核态） | I/O 耗时 |
| load / idle | CPU 负载、空闲（内核态） | 系统负载分析 |
| nweb | NWeb 模块 | ArkWeb 场景 |

经验法则：抓 UI 性能问题，`app ace graphic window ability ffrt sched freq multimodalinput` 是一组顺手的组合；不想挑就用二进制快照模式的默认集。tag 开得越多，缓冲区消耗越快，长时间采集要相应调大 `-b`。

**应用自己的打点**用 HiTraceMeter 接口（`startTrace`/`finishTrace`、`startAsyncTrace`/`finishAsyncTrace`），全部落在 app tag 下。成对使用是硬性要求——不同步配对会让可视化工具上的泳道显示错乱（出处：《HiTraceMeter 使用指导》）。API 19 起打点支持级别（D/I/C/M），用 `hitrace --trace_level I` 设置输出阈值，低于阈值的点不生效。

## 输出文件与打开方式

**文本格式**（默认）：trace 内容直接可读，每行形如：

```text
KstateRecvThrea-1132  ( 952) [003] .... 589942.951387: tracing_mark_write: B|952|H:CheckMsgFromNetlink|I62
```

字段依次是：线程名-线程ID、进程ID、CPU 核、时间戳（开机起算的秒）、事件类型。`B|pid|名称` 是切片开始，`E` 是结束，`C` 是计数器。文本格式适合 grep 快速确认打点是否生效，但不适合看全局。

**二进制格式**（.sys 文件）：默认保存在 `/data/log/hitrace/`。文件名规则：`trace_` 开头是快照模式产物、`record_trace_` 开头是录制模式产物，后面跟本地时间和 boot time——`trace_20250701215441@6016-653165227.sys` 表示 2025-07-01 21:54:41 生成，boot time 6016.653165227 秒。

拉取到本地后有两种打开方式：

```shell
hdc file recv /data/log/hitrace/trace_xxx.sys 本地目录
```

1. **SmartPerf-Host**：OpenHarmony 官方发行的可视化工具（gitcode.com/openharmony/developtools_smartperf_host 的 releases 下载），把 .sys 文件拖入即出泳道图，对鸿蒙打点的解析最完整，是首选。
2. **Perfetto UI**：二进制 trace 也可用 Perfetto 系工具打开；文本格式的 .ftrace 还可以直接在 DevEco Studio Profiler 会话区选 Open File 导入（出处：《HiTraceMeter 查看》）。

## 常见坑

官方文档列出的四个高频报错，原因和解法都明确（出处：《hitrace》常见问题）：

**`error: DumpSnapshot failed, errorCode(1)`**：hiview 进程状态异常。重启手机后重新采集。

**`error: xxx is not support category on this device`**：tag 名写错或设备不支持。先 `hitrace -l` 确认拼写。

**`errorCode(1004)`**：写文件失败。两个可能——`-o` 指定的路径不存在或无权限（文本格式 trace 建议一律写 /data/local/tmp）；或者磁盘空间不足，保证空闲大于 500MB 再采。

**`error: illegal path`**：二进制 trace（快照/录制模式）的 `-o` 路径必须是 /data/local/tmp 或其子目录，其他路径一律拒绝。

另外两个实战提醒：一是别忘了 `hitrace --trace_finish`。只 begin 不 finish，采集会一直挂着占缓冲区，影响后续采集和系统行为；二是采集本身有开销，buffer 越大、tag 越多开销越大，性能对比测试时优化前后的采集参数要完全一致，否则测出的差值可能来自工具而不是代码。

## 参考资料

- 《hitrace》（hitrace）
- 《HiTraceMeter 开发指导》（hitracemeter-guidelines-arkts / hitracemeter-guidelines-ndk）
- 《HiTraceMeter 查看》（hitracemeter-view）
- 《hiprofiler》（hiprofiler）
- 《hdc》（hdc）
