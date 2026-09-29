# 附录 B：常用 hdc / 系统命令

hdc 之于鸿蒙相当于 adb 之于 Android：主机侧命令行工具，负责设备连接、安装、文件传输与 shell 进入。进入设备 shell 后的 aa/bm/hilog/hitrace/hidumper 等是设备侧工具。以下按用途分组，命令以当前公开文档为准。

## 连接与设备管理

| 命令 | 用途 |
| --- | --- |
| `hdc list targets` | 列出已连接设备 |
| `hdc list targets -v` | 列出设备并显示连接方式（USB/TCP）与状态（Connected/Offline） |
| `hdc tconn <ip>:5555` | TCP 方式连接设备 |
| `hdc -t <connect-key> <command>` | 对指定设备执行命令（多设备时必用） |
| `hdc kill` / `hdc kill -r` | 停止/重启 hdc 服务，连接异常时的恢复手段 |
| `hdc shell` | 进入设备 shell |

## 安装与应用管理（bm）

| 命令 | 用途 |
| --- | --- |
| `hdc install <path/xx.hap>` | 推送并安装 HAP |
| `hdc shell bm install -p /data/local/tmp/x.hap` | 在设备侧安装 HAP |
| `bm install -p x.hap -r` | 覆盖安装 |
| `bm install -p x.hap -d` | 允许高版本覆盖为同包名低版本 |
| `bm install -p x.hap -g` | 安装 debug 签名应用时自动授予 user_grant 权限 |
| `bm install -s x.hsp` | 安装应用间共享库 |
| `bm install -p a.hap -s x.hsp y.hsp` | 同时安装应用与其依赖的共享库 |
| `bm uninstall -n <bundleName>` | 卸载应用 |
| `bm dump -n <bundleName>` | 查询应用包信息 |

## 文件传输

| 命令 | 用途 |
| --- | --- |
| `hdc file send <local> <remote>` | 推送文件到设备（常用目标：/data/local/tmp） |
| `hdc file recv <remote> <local>` | 从设备拉取文件（Trace、日志、快照） |

## 启动与控制应用（aa）

| 命令 | 用途 |
| --- | --- |
| `aa start -b <bundleName> -a <abilityName>` | 显式启动 UIAbility |
| `aa start -A <action> -U <uri>` | 隐式启动（按 action/uri） |
| `aa start -b .. -a .. -W` | 启动并测量耗时（API 20+）：冷启动输出系统侧收到请求到首帧绘制的 TotalTime，热启动输出到前台的耗时 |
| `aa force-stop <bundleName>` | 强制停止应用（冷启动测试前置步骤） |
| `aa process -b <bundleName> -a <ability> -p perf-cmd` | 应用调试/调优命令 |
| `aa dump -l` | 列出已运行的 Ability 信息 |

## 日志（hilog）

| 命令 | 用途 |
| --- | --- |
| `hilog` | 实时打印全量日志 |
| `hilog -T <tag>` / `hilog -D <domain>` | 按 tag/domain 过滤 |
| `hilog -L <D/I/W/E/F>` | 设置最低日志级别 |
| `hilog -r` | 清空日志缓冲 |
| `hilog -w start -t kmsg -f kmsglog -l 2M -n 100 -m zlib` | 开启 kmsg 落盘任务（文件数 100、单文件 2M、zlib 压缩） |
| `hilog -w stop` | 停止落盘任务 |

日志行格式：日期、时间戳、进程号、线程号、级别、domainID/进程名/tag、内容。

## 性能打点与抓取（hitrace）

| 命令 | 用途 |
| --- | --- |
| `hitrace -l` | 列出可用 tag（category） |
| `hitrace -t 10 -o /data/local/tmp/x.htrace app ace ability` | 按 tag 抓取 10 秒 Trace 到文件（默认 5 秒） |
| `hitrace --trace_begin app` / `--trace_finish` / `--trace_dump` | 手动控制抓取的起点、结束与导出 |
| `hitrace -b 32768 ...` | 设置 buffer 大小（KB，默认 18432） |
| `hitrace -z ...` | 压缩输出 |
| `hitrace --trace_level I` / `--get_level` | 设置/查询打点级别阈值（D/I/C/M） |
| `hitrace --record --trace_begin` | 长时采集任务模式 |
| `hitrace --start_bgsrv / --dump_bgsrv / --stop_bgsrv` | trace_service 快照模式的启停与导出 |

常用 tag：app（应用 HiTraceMeter 打点）、ace（ArkUI）、ability（Ability 框架）、graphic（图形）、freq、sched、disk、binder、ohos、window、render_service、zygote/appspawn（具体 tag 集以 `hitrace -l` 为准）。

## 系统信息导出（hidumper）

| 命令 | 用途 |
| --- | --- |
| `hidumper -s` | 列出全部系统服务（SA）及状态 |
| `hidumper -s <SAID> -a <args>` | 导出指定系统服务信息 |
| `hidumper --mem <pid>` | 进程内存详情（PSS 等） |
| `hidumper --cpuusage <pid>` | 进程 CPU 占用 |
| `hidumper -p <pid>` | 进程综合信息 |

## 性能采集（SP_daemon / hiprofiler / hiperf）

| 命令 | 用途 |
| --- | --- |
| `SP_daemon` | SmartPerf 采集守护：FPS、CPU、GPU、内存、功耗、温度等整机/应用级指标的命令行采集，配合 SmartPerf-Host 分析 |
| `hiprofiler_cmd -c - -o out.htrace -t 30` | hiprofiler 底层采集（ftrace-plugin 等，可指定 hitrace_categories），供深度性能分析 |
| `hiperf record` / `hiperf report` | CPU 采样与热点函数分析（perf 等价物） |

## 其它实用命令

| 命令 | 用途 |
| --- | --- |
| `param get <key>` / `param set <key> <v>` | 系统参数读写（如 bootevent 进度参数） |
| `param ls -r` | 列出系统参数 |
| `power-shell` | 设备电源状态转换（唤醒、休眠控制） |
| `uinput` | 模拟输入事件（自动化测试） |
| `mediatool recv all <dir>` | 导出媒体库文件 |
| `cem` | 公共事件管理 |
| `devicedebug` | 向调试应用发送信号 |
| `faultlog` / `/data/log/faultlog/` | 故障日志查看与存放路径 |
