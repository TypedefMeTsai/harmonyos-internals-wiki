# 6.1 应用沙箱与文件系统视图

## 为什么应用看不到真实文件系统

在鸿蒙设备上 `ls /data`，会看到完整的物理目录树；但应用代码里打开 `/data/...` 路径，要么失败，要么读到的不是同一个东西。这是应用沙箱在工作：官方定义是——应用沙箱是一种以安全防护为目的的隔离机制，避免数据受到恶意路径穿越访问；在沙箱保护机制下，应用可见的目录范围即为"应用沙箱目录"，所有应用的目录可见范围均经过权限隔离与文件路径挂载隔离，形成独立的路径视图，屏蔽了实际物理路径。

实现上，每个应用有独立的 uid（官方文档：uid 是系统中用于应用沙箱隔离的唯一标识，分配给每个应用进程，确保应用间文件系统、内存空间等相互隔离），再通过挂载命名空间把物理路径映射成应用视角的沙箱路径。两套路径的对应关系（官方 IDE 文档给出的物理位置）：

```
沙箱路径（应用视角）                    物理路径（设备视角）
/data/storage/el2/base/...      →      /data/app/el2/100/base/{packageName}/...
/data/storage/el2/database/...  →      /data/app/el2/100/database/{packageName}/...
```

开发调试时在 DevEco Studio 的 Device File Browser 里能看到两种视图："普通文件视图"按真实物理路径显示，应用沙箱目录就在上面的物理路径下。应用代码里**只能用沙箱路径**——硬编码物理路径是移植性事故，路径里的 userId（100 是当前用户）和包名组织方式都可能变。

## 三类文件：应用文件、用户文件、系统文件

官方应用沙箱目录文档把应用可访问的文件范围分成三类，访问方式完全不同：

**应用文件**：应用自己的数据，放在应用文件目录下，随应用卸载删除（模块级路径随 HAP/HSP 卸载删除，应用级路径在全部模块卸载后删除）。应用可以自由读写，不需要任何权限。应用文件目录下按用途预分了子目录，通过 Context 获取：

```typescript
let ctx = this.getUIContext().getHostContext();
let filesDir = ctx.filesDir;          // 持久文件
let cacheDir = ctx.cacheDir;          // 缓存（系统可在空间不足时清理）
let tempDir = ctx.tempDir;            // 临时文件
let preferencesDir = ctx.preferencesDir;  // 用户首选项持久化文件
let databaseDir = ctx.databaseDir;    // 数据库
let distributedFilesDir = ctx.distributedFilesDir;  // 分布式文件
```

目录用途的纪律在官方存储规范文档里写成了"推荐/不得"条款：按业务维度在 cacheDir、filesDir 下建子目录；**不得**把缓存或临时文件放进 filesDir、databaseDir、preferencesDir 等非缓存目录；**不得**把用户手动保存、下载的内容放在 cacheDir 或 tempDir——缓存目录系统会清，用户内容放丢了就是事故。

**用户文件**：照片、视频、文档这些用户数据，归用户所有，应用**不能用文件路径直接访问**。要通过系统提供的框架：Picker（用户选文件）、photoAccessHelper（相册）、AudioViewPicker 等，拿到的是授权后的 URI 句柄。向公共目录写入的内容应当是用户主动保存的——这是规范也是审核要求。

**系统文件**：应用运行必需的少量系统文件以只读方式映射进沙箱视图，应用读得到、改不了。

## 加密等级：el1 到 el5

沙箱路径里的 `el2` 是加密等级（Encryption Level）。设备锁屏状态下不同等级的目录可访问性不同：el1 目录设备开机后即可访问，el2 目录需要设备至少解锁一次后才可访问。这直接解释了备份恢复、锁屏后台任务场景的目录选择——官方数据迁移文档里把 APK 的 `/data/user_de/`（device encrypted）映射到鸿蒙 el1 的备份恢复路径，正是这个等级语义。更高等级（el3/el4/el5）用于更敏感的数据分区[待验证：el3–el5 的具体解锁语义与开放范围]。

给 Context 换区的代码很简单，但影响的是之后所有路径解析：

```typescript
import { contextConstant } from '@kit.AbilityKit';
ctx.area = contextConstant.AreaMode.EL1;  // 之后 filesDir 等指向 el1 分区
```

锁屏期间需要读写文件的后台任务（比如代理提醒触发后要记日志），必须把对应目录切到 el1，否则拿到的是不可访问错误。

## Native 侧的沙箱路径

NDK 开发有两条官方路径（Core File Kit 文档）：一是 ArkTS 侧取好沙箱路径传给 Native（推荐，路径来源唯一）；二是 Native 侧按"沙箱路径与真实物理路径对应关系"自行拼接——能不用就不用，拼接意味着把目录结构的假设写死进了 so。

## 文件 I/O 的性能要点

沙箱解决"放哪"，性能解决"怎么读写"。几条规则（第 22 章会展开成案例）：

1. **主线程只用异步 API**。`fs.readText` 走系统 I/O 线程池（5.3 节），`readTextSync` 让主线程亲自读盘。冷启动路径上同步读一个几百 KB 的配置文件，就是几十毫秒的启动时延。
2. **小文件合并，大文件流式**。大量小文件逐个 open/read/close 的系统调用开销远超大文件；大文件用 `fs.createRandomAccessFile` 或流接口分块读，别一次性 read 进内存。
3. **缓存目录主动治理**。cacheDir 系统会清，但清理时机不可控；应用自己该有上限和淘汰策略，否则"应用越用越大"的差评就来了。
4. **分布式文件目录的特殊性**：distributedFilesDir 下的文件在组网下对端可见，一端修改另一端"立即"可见（速度取决于网络），设备离线后远端数据不再呈现——官方分布式文件系统文档提醒离线感知有延迟，部分消息有约 4s 超时，访问远端文件要做超时容错。

## 实际排查

- **文件找不到**：先确认用的是 Context 给的沙箱路径还是手拼路径；再确认加密等级（锁屏场景访问 el2 必失败）。
- **空间占用核查**：`hdc shell "du -sh /data/app/el2/100/base/{packageName}/*"` 按目录看占用，对照缓存治理逻辑找膨胀点。
- **权限类报错**：读用户文件报权限错误时，方向不是申请文件权限，而是改用 Picker/photoAccessHelper——用户文件的授权模型是"用户选定"，不是"应用声明"。

## 版本演进

- API 9–11：沙箱目录结构与 Context 路径接口定型，el1/el2 分区可用；
- API 12（5.0）：单框架后用户文件访问框架（Core File Kit）成为唯一通道，路径直访用户数据彻底关闭；备份恢复与 APK 迁移的目录映射关系文档化；
- API 20+（6.0）：存储使用规范（推荐/不得条款）、Device File Browser 双视图等配套完善；云同步（全维度同步上云、版本管理）进入文件服务。

## 参考资料

- 华为开发者文档：《应用沙箱目录》（app-sandbox-directory：沙箱定义、三类文件、挂载隔离）
- 华为开发者文档：《应用上下文（Stage 模型）》（application-context-stage：Module 级与应用级路径、卸载语义）
- 华为开发者文档：《Native 侧文件访问》（file-native-side：两种获取沙箱路径的方案）
- 华为开发者文档：《IDE Device File Browser》（ide-device-file-explorer：物理路径 /data/app/{el1,el2}/100/{base,database}/{packageName}）
- 华为开发者文档：《存储使用规范》（storage-usage-file-lifecycle：推荐/不得条款）
- 华为开发者文档：《应用数据迁移适配》（app-data-migration-adaptation：el1 与备份恢复路径）
- 华为开发者文档：《分布式文件系统概述》（distributed-fs-overview：4s 超时与冲突重命名）
- OpenHarmony 仓库 `openharmony/filemanagement_app_file_service`、`openharmony/filemanagement_user_file_service`[待验证：仓库名精确拼写]
