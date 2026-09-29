# 12.1 网络栈与请求性能

## 两套 HTTP 客户端怎么选

鸿蒙应用发起 HTTP 请求有两条官方路径：

**@ohos.net.http**：NetStack 提供的传统 HTTP 模块（SystemCapability.Communication.NetStack），API 6 起可用。每个 HttpRequest 对象对应一次请求，支持常见方法（GET/POST/PUT/DELETE 等）、请求头定制、超时配置。两个超时参数要分清（出处：《http 请求》）：`readTimeout` 是从请求开始到结束的总时间（含 DNS、建连、传输），默认 60000ms；`connectTimeout` 仅连接建立阶段，默认 60000ms。

**rcp（RemoteCommunicationKit）**：远程通信平台（SystemCapability.Collaboration.RemoteCommunication），API 11（4.1.0）起可用，通过 `rcp.createSession()` 创建会话、`session.fetch(request)` 发起请求。相对 http 模块，rcp 的性能相关能力明显更强：

- **会话级配置**：超时、缓存、DNS、追踪统一在 Session/Request 的 configuration 里管理，会话天然支持连接复用；
- **timeInfo 耗时追踪**：把请求各阶段耗时结构化返回（下文详述）；
- **自定义 DNS**：dnsRules 允许应用接管域名解析；
- **HTTP 缓存**：会话级响应缓存，命中时零网络耗时；
- **细粒度事件**：onDownloadProgress、onDataEnd 等回调，便于大文件场景的进度与中断管理。

选型建议：新代码、对性能与可观测性有要求的场景用 rcp；维护老代码或做极简请求时 http 仍然够用。第 22 章会基于两者做缓存与弱网的工程实践，本章只讲协议栈层面的事实。

## 把一次请求的时间拆开：timeInfo 与 httpPhase

网络性能分析的第一步是把"总耗时"拆成阶段。rcp 提供了开箱即用的手段：请求配置中打开 `tracing.collectTimeInfo`，响应里就带 timeInfo（出处：《rcp 请求耗时追踪》）：

```typescript
const configuration: rcp.Configuration = {
  tracing: { collectTimeInfo: true }
};
const request = new rcp.Request('https://www.example.com', 'GET');
request.configuration = configuration;

session.fetch(request).then((response: rcp.Response) => {
  const info = response.timeInfo!;
  // 各阶段耗时由时间戳差值算出
  const tlsTime = info.tlsHandshakeTimeMs - info.connectTimeMs;       // TLS 握手（不含建连）
  const firstPackage = info.startTransferTimeMs - info.preTransferTimeMs; // 首包耗时
  const remainder = info.totalTimeMs - info.startTransferTimeMs;      // 剩余数据接收
});
```

三个指标的含义：`tlsTime` 反映 TLS 协商成本（会话复用可以压低它）；`firstPackage` 近似服务器处理 + 网络 RTT，是服务端慢的探针；`remainder` 大说明响应体大或带宽受限。DNS 和 TCP 建连耗时同样可从 timeInfo 字段差值推出。拿到这张分解表，"网络慢"立刻变成可讨论的具体问题：慢在解析、慢在握手、慢在服务端、还是慢在传输。

请求失败时，rcp 错误信息中的 **httpPhase** 字段用六位数标记失败发生的阶段，每位对应 dns/tcp/tls/snd/rcv/bdy（出处：《rcp 错误字段》）：

| httpPhase | 含义 |
| --- | --- |
| 100000 | 失败在 DNS 阶段 |
| 110000 | 失败在 TCP 阶段 |
| 111000 | 失败在 TLS 阶段 |

失败定位从阶段开始：DNS 阶段失败查域名与网络可用性，TCP 阶段查连通性与运营商，TLS 阶段查证书与时间配置——比对着一个笼统的"请求失败"猜要高效得多。

## 连接复用、DNS 与缓存：客户端的三大杠杆

**连接复用与超时**。TCP+TLS 握手的合计成本在弱网下可达数百毫秒，每请求新建连接是巨大的浪费。rcp 的 Session 默认维护连接，应用应当"长会话、多请求"，而不是每个请求建一个 Session。配合预连接思路：应用启动或空闲时对核心域名提前建连，把 DNS 查询和 TCP/TLS 握手成本从用户点击的关键路径上挪走——这正是第 8 章完成时延优化中"网络请求提前发送"建议的底层依据。超时配置要有区分：connectMs 给短（如 6s，快速失败重试），transferMs 给足（如 60s，容忍大文件与弱网），官方重连示例就是 `connectMs: 6000, transferMs: 60000`（出处：《应用网络重连》）。

**自定义 DNS**。rcp 的 dnsRules 允许应用按域名返回指定 IP 列表：

```typescript
request.configuration = {
  dns: {
    dnsRules: (host: string, port: number): rcp.IpAddress[] => {
      if (host === 'example.com') {
        return ['192.168.1.1', '192.168.1.2'];
      }
      return [];
    }
  }
};
```

用途有二：一是内置 HTTPDNS 结果，绕过本地 DNS 的污染与慢解析；二是配合服务端多线部署做就近接入。注意返回空数组表示该域名走系统默认解析，别把兜底逻辑漏掉。

**HTTP 缓存**。rcp 会话可配置响应缓存：第一次请求从网络获取并写入缓存，后续相同请求命中缓存直接返回（出处：《rcp 缓存》）。对遵循 HTTP 缓存协议的静态资源（图片、配置、脚本），这是零成本的时延优化。业务数据的缓存策略（先出缓存再二刷）属于架构设计，在第 22 章展开。

## 弱网：从感知到应对

弱网环境的残酷之处在于超时阈值变得不可靠：Wi-Fi 下 2 秒的请求，电梯里可能 20 秒也回不来。应用要么被动等超时，要么主动感知网络状态并调整行为。鸿蒙把感知能力收在 Network Boost Kit：

**网络质量与场景识别**（出处：《Network Boost Kit 指南》《网络导航》）：

```typescript
import { netQuality } from '@kit.NetworkBoostKit';

netQuality.on('netSceneChange', (list: Array<netQuality.NetworkScene>) => {
  list.forEach((sceneInfo) => {
    if (sceneInfo.scene == 'congestion') {
      // 检测到网络拥塞：降码率、降分辨率、延迟非关键请求
    }
    if (sceneInfo.scene == 'normal') {
      // 恢复
    }
    if (sceneInfo.weakSignalPrediction) {
      // 弱信号预测：网络即将变差，提前增大缓存、暂停预加载
    }
  });
});
```

`netQuality.on('netQosChange')` 则提供更细的信号强度、下载速度、网络时延数据。系统还提供了反向通道：应用可以通过传输体验反馈接口告诉系统"我现在体验差"，触发系统侧网络加速（出处：《Network Boost C API 概述》）。

**连接迁移**。弱网时系统可能发起多网迁移（Wi-Fi↔蜂窝、主卡↔副卡）。迁移期间旧连接会被掐断，应用在收到迁移通知后按建议重建连接，能把"断网十几秒"压缩成一次快速重连（出处：《Network Boost C API 概述》）。

**网络连接管理**（`@ohos.net.connection`）是更基础的兜底：`getDefaultNetSync()` 获取当前默认网络，`getNetCapabilities` 查网络类型与能力，用于"无网不请求、省流模式降质量"这类基础判断（出处：《网络连接管理》）。

弱网应对的总原则：感知 → 降级 → 恢复。感知靠 netQuality；降级手段包括降低媒体质量、合并与延后非关键请求、增大播放缓冲；恢复要在 scene 回到 normal 后主动做，别等下一次请求失败。

## 常见问题与误区

**"网络慢是服务端的事"。** timeInfo 拆解后经常会发现慢在客户端能管的环节：DNS 走了慢通道（自定义 DNS 解决）、每请求重新握手（会话复用解决）、请求发出太晚（第 8 章的两个反模式）。先拆时间，再决定怪谁。

**"超时长一点更稳"。** 超长超时在弱网下的表现是"长时间转圈"，用户早走了。合理策略是短 connect 超时快速失败 + 有限次重试 + 明确的失败 UI，官方示例的重试间隔为 2 秒并限定重试次数。

**"http 和 rcp 混着用无所谓"。** 两套客户端各自维护连接池，混用意味着同一域名建两份连接，既浪费也无法统一观测。选定一套为主。

## 参考资料

- 《rcp 远程通信平台》（remote-communication-rcp）
- 《rcp 请求耗时追踪》（remote-communication-tpms）
- 《rcp 错误字段》（remote-communication-error-field）
- 《rcp 自定义 DNS》（remote-communication-customdnsconfig / remote-communication-cpo）
- 《rcp 缓存》（remote-communication-cache-basic）
- 《http 数据请求》（js-apis-http / http-request）
- 《应用网络重连》（application-network-reconnection）
- 《Network Boost Kit 开发指南》（network-boost-kit-guide / network-navigator）
- 《网络连接管理》（net-connection-manager）
