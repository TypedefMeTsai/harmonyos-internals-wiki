# 15.1 hilog 体系与高效查日志

## 这个工具解决什么问题

应用出问题时，Trace 能告诉你耗时分布，但"业务逻辑走到了哪一步、哪一步开始不对劲"要靠日志。hilog 是鸿蒙统一的日志系统：系统框架、系统服务、应用都通过 HiLog 接口往里写，开发者通过 `hdc shell hilog` 命令读取，或在 DevEco Studio 的 Log 窗口查看。日志还会落盘到设备 `/data/log/hilog` 目录，可导出后离线分析（出处：《hilog》）。

会用 hilog 的标志不是会敲 `hdc shell hilog`——那是把整机日志往终端上倒，几秒就被淹没。高效的用法是带着过滤条件去查：只看我的进程、只看我的 tag、只看 ERROR 以上。本节把过滤体系和那些"日志为什么没了"的坑讲清楚。

## 三层结构：domain、tag、level

每条 hilog 日志带三个归属信息：

- **level**：DEBUG / INFO / WARN / ERROR / FATAL 五级，行内显示首字母（D/I/W/E/F）。
- **domain**：模块域编号，十六进制，系统给每个子系统分配了固定段（如 0x3200 段属于某类系统服务）。应用日志一般用 0x0000 段或自定义 domain。
- **tag**：模块自定义字符串，最长 31 字节，超出从末尾截断；不建议用中文，可能乱码或对不齐。

日志行格式七列（出处：《hilog》）：

```text
04-19 17:02:14.735  5394  5394  I  A03200/test_server/testTag:  this is a info level hilog
日期   时间          pid   tid   级别 domainID/进程名/tag:      内容
```

domainID 列的前缀字母有含义：`A03200` 中 A 表示应用日志（LOG_APP），3200 是 domainID；`C01406` 中 C 表示系统日志（LOG_CORE）。进程名最长 30 字节，超长从开头截断保留尾部。

**log 版本与 nolog 版本**：商用设备分两种。"设置 → 关于本机 → 软件版本"中版本号以 log 结尾的是 log 版本（如 `5.0.0.36(xxxlog)`），hilog 默认开启落盘、全局级别默认 INFO；nolog 版本默认不打印日志，打开开发者模式后 API 15 起全局级别为 WARN（API 14 及之前为 INFO）。连上 DevEco Studio 5.0.4 及之后版本时全局级别为 INFO，断开 IDE 重启后恢复 WARN（出处：《hilog》）。在 nolog 设备上"看不到 DEBUG/INFO 日志"是设计如此，不是设备坏了。

## 高效过滤：常用命令组合

查日志的基本功是把过滤参数叠起来（出处：《hilog》命令表）：

```shell
hilog -x                       # 非阻塞读一次缓冲区就退出（脚本里必用，缺省是阻塞跟随）
hilog -L E                     # 只看 ERROR 及以上
hilog -t app                   # 只看应用日志（还有 core/init/kmsg）
hilog -D 01B06                 # 按 domain 过滤
hilog -T SAMGR                 # 按 tag 过滤
hilog -P 618                   # 按进程号过滤
hilog -e "AppFreeze"           # 消息内容正则匹配
hilog -v year -v msec          # 时间格式：加年份、毫秒精度
```

实战中最高频的组合：

```shell
# 看某个应用进程的 WARN 以上日志
hilog -P $(pidof com.example.app) -L W

# 抓崩溃现场：ERROR 级别 + 关键字
hilog -L E -e "com.example.app"

# 配合 faultlog 时间窗：先清缓冲区，复现后一次倒出
hilog -r            # 清空缓冲区
# ……设备上复现问题……
hilog -x > /tmp/scene.log
```

`hilog -r`（清缓冲区）+ 复现 + `hilog -x`（倒出）是抓干净现场的标准三步，省去事后在海量历史日志里找时间窗。

缓冲区管理：`hilog -g` 查各类型缓冲区大小（常见 16MB），`hilog -G 16M` 改大（上限 16MB）。日志级别控制：`hilog -b E` 设全局最低级别，`hilog -b D -D 0x3200` 只给某个 domain 开 DEBUG，`--persist` 让设置重启后保留。排查"别的模块日志太吵淹没了我的"时，可以 `hilog -b X` 全关，再单独开自己 domain 的级别。

**落盘任务**：log 版本默认落盘；nolog 版本需手动开。`hilog -w query` 查任务，`hilog -w start -n 1000` 开启（-f 文件名、-l 单文件大小、-n 文件数、-m 压缩算法 zlib/zstd/none），`hilog -w stop` 停止，`hilog -w clear` 删除落盘文件。落盘文件在 `/data/log/hilog`，用 hilogtool 解析导出。

## 日志为什么丢了：超限机制

hilog 有明确的流量治理，超出规格的日志直接丢弃。三种丢失形态，日志里各有维测关键字（出处：《hilog》）：

**LOGLIMIT**：进程或 domain 打印超限被管控。应用日志按进程管控（pid 维度），系统日志按 domain 管控。提示形如：

```text
W A00032/com.example.myapplication/LOGLIMIT: ==com.example.myapplication LOGS OVER PROC QUOTA, 3091 DROPPED==
```

含义：该进程在提示时间点前有 3091 行被丢弃。官方建议：**每个应用进程每秒打印日志量不超过 50KB**，每个 domainID 每秒不超过 50KB。可用 `hilog -Q pidoff` / `hilog -Q domainoff` 关闭管控（debug 应用默认关闭），但治本还是精简日志——高频回调里打日志是最常见的超限原因，也是第 7 章提到的主线程耗时来源之一。

**Slow reader missed**：日志产生太快，缓冲区里没来得及读走的部分被循环覆盖。提示形如 `========Slow reader missed log lines: 137`。处置：`hilog -g` 查大小、`hilog -G 16M` 扩到最大，并找出疯狂刷日志的模块关掉。

**write socket failed**：进程写日志 socket 失败丢包。原因除了日志量太大，还可能是系统高负载（CPU 高占用或低内存导致 hilogd 处理不过来）。处置同上：关闭其他领域日志（`hilog -b X`），只开本模块。

用正则 `LOGLIMIT|Slow reader missed|write socket failed` 一条命令扫描日志里所有丢失事件。官方强调：出现这些提示说明日志已经丢失、无法找回，只能本地复现重抓。

日志统计：`param set persist.sys.hilog.stats true` 后重启，`hilog -s` 输出统计报告——各级别行数占比、各 domain 的行数/字节数/DROPPED 数、每秒最高打印频率（MAX_FREQ）。想给应用的日志量做体检，这是最直接的工具。

## 性能开销

日志不是免费的：每次调用要走格式化、写 socket、落盘（如果开启）。高频路径（每帧回调、列表绑定）里的日志既拖慢主线程，又会触发超限丢日志，还可能在性能测试里污染数据。实践原则：

- 高频路径用 DEBUG 级并确保 release 包不输出，或干脆删除；
- 关键路径的一次性事件（启动、故障）用 INFO；
- 循环里打日志前先想想第 8 章那个 aboutToAppear 里 for 循环 console.debug 打出完成时延超标的官方案例。

## 参考资料

- 《hilog》（hilog）
- 《hilogtool》（hilog-tool）
- 《DevEco Studio 日志分析》（ide-setup-hilog）
- 《HiLog API 参考》（js-apis-hilog / capi-log-h）
