# 6.2 Preferences、KV Store 与关系型数据库

## 三套机制，三个量级

鸿蒙给结构化持久化准备了三套机制，对应三种数据量级：

- **Preferences（用户首选项）**：轻量级键值对，存配置——字体大小、夜间模式开关、登录态标记；
- **KV-Store（键值型数据库）**：单条记录较大、需要跨设备同步的键值数据，来自分布式数据服务；
- **RDB（关系型数据库）**：有结构、有查询条件、有事务的正经数据库，聊天记录、订单、离线缓存表。

选错的代价很具体：往 Preferences 里塞大对象，启动时全量加载直接把冷启动拖慢；把高频写入的配置项放进 RDB，每次写都过一遍 SQL 引擎；该同步的数据用了本地 Preferences，换设备就丢。下面把每套机制的工作方式和性能脾气讲清楚。

## Preferences：全量加载内存的 KV

官方定义：用户首选项为应用提供 Key-Value 键值型数据处理能力，支持持久化轻量级数据；键为字符串，值可为数字、字符串、布尔、数组、Uint8Array、object、bigint。持久化文件在 `preferencesDir` 下。

机制上最要紧的一句话写在官方术语表里：**数据以文本形式保存，应用使用时全量加载到内存**。这决定了它的性能模型：

- **读**：内存查找，纳秒到微秒级，随便读；
- **写**：`put` 改的是内存，`flush()` 才落盘。落盘是全量序列化写文件——文件越大，flush 越贵；
- **加载**：`getPreferences` 时整个文件读进内存并解析，文件越大启动越慢。

```typescript
import { preferences } from '@kit.ArkData';

let prefs = preferences.getPreferencesSync(ctx, { name: 'settings' });
prefs.putSync('fontSize', 16);
prefs.flush();  // 显式落盘；不 flush 进程异常退出就丢
```

由此推出使用纪律：**单实例数据量控制在 KB 级、键数量几十个以内**；大 value（长 JSON、Base64 图片）挪到文件或数据库；flush 别放在高频路径上逐条刷，攒一批再刷。

Preferences 还有两种存储模式的区分（官方术语表）：**XML 存储模式**（默认，通用性强、支持跨平台，不支持多进程并发）和 **GSKV 存储模式**（支持多进程并发场景）。多进程访问同一个首选项文件，要么切 GSKV，要么用 sendablePreferences（共享用户首选项，支持在多线程/并发实例间安全访问的变体）。排查"另一个进程改的没生效"先看模式对不对。调试侧，HarmonyOS 6.0 起官方提供了 preferences 数据库调试工具，可以直接查看设备上 preference_xml 与 preference_kv 数据库的内容（元数据和用户数据），不用再把文件 pull 出来猜。

## KV-Store：为分布式而生的键值库

KV-Store 属于分布式数据服务（ArkData），定位不是"本地配置"，而是**单设备存储 + 多设备协同**。创建时通过 kvManager 拿 store，Schema 可选，安全等级（S1–S4）可选：

```typescript
import { distributedKVStore } from '@kit.ArkData';

let kvManager = distributedKVStore.createKVManager(config);
let options: distributedKVStore.Options = {
  createIfMissing: true,
  encrypt: false,
  backup: true,
  autoSync: false,   // 官方建议验证同步功能时先关自动同步，手动调 sync
  kvStoreType: distributedKVStore.KVStoreType.SINGLE_VERSION,  // 单版本；协同用 DEVICE_COLLABORATION
  securityLevel: distributedKVStore.SecurityLevel.S3
};
let store = await kvManager.getKVStore('storeId', options);
await store.put('theme', JSON.stringify({ dark: true }));
```

分布式同步的机制（官方 KV-Store 同步文档）：底层通信组件完成设备发现和认证后通知上层设备上线，数据管理服务在两台设备间建立**加密数据传输通道**，之后通过 put/delete 触发自动端端同步——应用更新数据后，分布式数据库自动把本端数据推送到远端，同时把远端数据拉到本端，不需要主动调 sync（autoSync 开启时）。

性能与限制要点：键值对的值有大小上限（Entry 的 value 字段标注了 MAX_VALUE_LENGTH，Uint8Array/string 类型受该上限约束）；同步走软总线（1.5 节），弱网或设备离线时同步有延迟和失败，关键数据别指望"写了立刻对端可见"；协同库（DEVICE_COLLABORATION）和单设备库（SINGLE_VERSION）行为不同，建库时选错后面全是坑。

## RDB：带谓词和事务的关系库

关系型数据库（relationalStore，`@kit.ArkData`）面向结构化数据：表结构、索引、事务、用谓词（RdbPredicates）表达查询条件，不用手写 SQL 字符串（也支持原生 SQL）。标准用法：

```typescript
import { relationalStore } from '@kit.ArkData';

let store = await relationalStore.getRdbStore(ctx, {
  name: 'app.db', securityLevel: relationalStore.SecurityLevel.S1
});
// 建表（version 升级时在 onUpgrade 里迁移）
store.executeSql('CREATE TABLE IF NOT EXISTS note (id INTEGER PRIMARY KEY, title TEXT, ts INTEGER)');

let predicates = new relationalStore.RdbPredicates('note');
predicates.greaterThan('ts', yesterday);
let resultSet = await store.query(predicates, ['id', 'title']);
```

RDB 同样支持分布式能力：把表设为分布式表后，可向数据管理服务发起同步（推送或拉取两种方式），组网内其他设备的数据变化同步回本端时触发已注册的回调（官方 RDB 同步文档）。

性能上的纪律：

- **主线程禁同步查询**。getRdbStore、query、insert 都有异步形式，全走系统 I/O 线程池；列表页加载时的同步建库 + 全表查询是启动卡顿的经典根因；
- **ResultSet 用完关**。游标不关是句柄泄漏，官方资源泄漏检测盯的就是这类资源；
- **批量写用事务**。逐条 insert 每条一次磁盘同步，包进事务一次同步，差一个数量级；
- **索引照查询条件建**。谓词走的字段没索引就是全表扫描。

## 选型速查

| 数据特征 | 选择 |
| --- | --- |
| 几十个键以内的应用配置 | Preferences |
| 多进程/多线程都要访问的配置 | Preferences（GSKV）或 sendablePreferences |
| 需要跨设备同步的键值数据 | KV-Store（DEVICE_COLLABORATION） |
| 结构化、要条件查询、要事务 | RDB |
| 大对象（图片、音视频、长文本） | 文件（沙箱目录），库里只存路径 |
| 需要跨设备同步的结构化表 | RDB 分布式表 |

## 实际排查

- **启动慢且用了 Preferences**：看 preferencesDir 下文件大小，超过几百 KB 就要拆；DevEco Profiler Launch 会话里找 getPreferences 的加载区间。
- **主线程卡顿伴随数据库操作**：Frame/CPU 会话里主线程出现 RDB 相关长区间，改异步 + 事务 + 索引三板斧。
- **跨设备数据没同步**：先查组网（1.5 节的 hidumper -s 4700 命令），再查 autoSync 设置和安全等级，最后看 sync 调用的错误回调。
- **Preferences 内容核对**：HarmonyOS 6.0+ 用官方 preferences 调试工具直读 preference_xml/preference_kv 内容。

## 版本演进

- API 9–11：Preferences、KV-Store、RDB 三套 API 随 Stage 模型齐备；
- API 12（5.0）：单框架后 ArkData 整合为统一数据管理 Kit，分布式数据服务同步模型文档化；
- API 18+：sendablePreferences 提供多线程安全的共享首选项；HarmonyOS 6.0 起 preferences 调试工具可用；数据加密、按设备与数据等级的访问控制（S1–S4 安全等级）成为标配能力。

## 参考资料

- 华为开发者文档：《用户首选项概述》（data-persistence-by-preferences）与 API 参考 `js-apis-data-preferences`（值类型、preferencesDir）
- 华为开发者文档：《数据管理术语表》（data-terminology：全量加载内存、XML/GSKV 两种存储模式）
- 华为开发者文档：《Preferences 调试工具》（preferences-debug-tool：HarmonyOS 6.0 起，preference_kv/preference_xml）
- 华为开发者文档：`js-apis-data-sendablepreferences`（共享用户首选项）
- 华为开发者文档：《KV-Store 数据同步》（data-sync-of-kv-store：加密通道、自动端端同步）、《RDB 数据同步》（data-sync-of-rdb-store：推送/拉取两种同步触发）
- 华为开发者文档：`js-apis-distributedkvstore`（Entry、MAX_VALUE_LENGTH、SecurityLevel、kvStoreType）
- 华为开发者文档：《按设备与数据等级访问控制》（access-control-by-device-and-data-level：Schema、encrypt、S3 配置示例）
- OpenHarmony 仓库 `openharmony/distributeddatamgr_kv_store`、`openharmony/distributeddatamgr_relational_store`、`openharmony/distributeddatamgr_preferences`
