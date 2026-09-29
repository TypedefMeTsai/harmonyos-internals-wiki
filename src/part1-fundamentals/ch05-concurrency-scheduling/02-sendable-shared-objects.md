# 5.2 Sendable 共享对象与线程间通信

## Actor 模型的软肋：大对象怎么传

5.1 节说过，ArkTS 并发实例之间默认靠序列化传对象，100KB 数据约 1ms。这个开销有两个硬伤：

- **大数据传不起**。解码一张 2000×2000 的图片，像素数据十几 MB，序列化一次就是上百毫秒——跨线程传图如果不换方案，并发带来的收益全被传输吃掉。
- **带行为的对象传不了**。序列化只搬数据，class 实例的方法、原型链全丢，到对端变成一具"数据尸体"。想在两个线程共享一个带方法的对象（比如一个带查询接口的缓存），默认模型做不到。

**Sendable 协议**就是为这两个场景设计的。官方定义：Sendable 协议定义了 ArkTS 的可共享对象体系及规格约束，符合 Sendable 协议的数据可以在 ArkTS 并发实例间传递；**默认采用引用传递方式**，同时也支持拷贝传递方式。官方 Sendable 指南把适用场景说得很直接：跨并发实例传输大数据（100KB 以上）、跨并发实例传递带方法的 class 实例对象。

## @Sendable class：让对象可共享

让一个类可共享，用 `@Sendable` 装饰：

```typescript
@Sendable
export class ImageCache {
  private store: Map<string, ArrayBuffer> = new Map();

  put(key: string, data: ArrayBuffer): void {
    this.store.set(key, data);
  }
  get(key: string): ArrayBuffer | undefined {
    return this.store.get(key);
  }
}
```

这个类的实例现在可以直接作为 taskpool 任务参数或 Worker 消息传递：对端拿到的是**同一个对象的引用**，方法可用，改动互见。官方 Native 互操作文档对传递方式的描述值得逐字记住：Sendable 共享对象序列化不会产生另外一份拷贝数据，而是直接传递对象引用到反序列化线程，性能上相比非 Sendable 对象的序列化和反序列化更高效。

引用传递能成立，是因为运行时为 Sendable 对象准备了一片**共享堆**，并发实例的虚拟机都能访问它——这是对"每个并发实例独立堆"规则的受控开口。开口受控，体现在一组硬约束上（官方约束文档）：

- Sendable 对象的**布局不可变**：禁止增删属性、改变属性类型（放进 TS/JS 容器或从 TS/JS 接口拿到后同样禁止）；
- Sendable class 只能继承 Sendable class，属性只能是 Sendable 兼容类型（基础类型、其他 Sendable 类、collections 里的 Sendable 容器等）；
- 想在子线程用 `instanceof A` 判断成立，定义类 A 的 ets 文件必须用 `"use shared"` 指令标记为**共享模块**——官方并发 FAQ 里"传过去后 instanceof 返回 false"的经典问题，根因就是漏了这条；
- 跨线程访问 Sendable 对象有同步开销，热路径上频繁读写的共享对象要评估收益。

## 共享之后的竞争：AsyncLock

引用传递把 Actor 模型的安全网撤掉了一半——两个线程同时改一个 Sendable 对象，数据竞争回来了。官方给的配套工具是 **AsyncLock**（异步锁，`@kit.ArkTS` 里的 `lock.AsyncLock`）：

```typescript
import { lock } from '@kit.ArkTS';

@Sendable
export class Counter {
  private count: number = 0;
  private locker: lock.AsyncLock = new lock.AsyncLock();

  async increment(): Promise<number> {
    return this.locker.lockAsync(() => {   // 锁内是异步临界区
      this.count++;
      return this.count;
    });
  }
}
```

AsyncLock 与经典互斥锁的差别在于它是异步的：拿不到锁的任务挂起等待而不是阻塞线程，避免把并发线程堵死。但原则不变——**锁的粒度要小，临界区要短**，否则共享带来的并发度又被锁竞争吃回去。最佳实践是优先用"分片 + 归并"（各线程处理自己那份数据，最后合并）减少共享面，必须共享时才上锁。

## 跨线程的 Context：SendableContextManager

还有一类常见需求：子线程里要用到主线程的 Context（比如 eventHub 发事件回主线程、访问应用资源）。普通 Context 不是 Sendable，传不过去。`sendableContextManager`（`@kit.AbilityKit`）提供转换通道：

```typescript
import { sendableContextManager } from '@kit.AbilityKit';

// 主线程：开启 eventHub 多线程支持，转换出 SendableContext
sendableContextManager.setEventHubMultithreadingEnabled(context, true);
let sendableContext = sendableContextManager.convertFromContext(context);

// 包装进 @Sendable 对象，传给 Worker
let obj = new SendableObject(sendableContext, 'BaseContext');
worker.postMessageWithSharedSendable(obj);
```

官方 API 参考里的完整示例演示了整条链：Worker 侧拿到 SendableContext 后通过 eventHub 触发主线程注册的事件，实现跨线程通信。这把"Context 不能跨线程"的老限制打开了一个标准化的口子。

## 性能决策：什么时候值得上 Sendable

把账算清楚再动手：

- **小对象（几 KB 以内）**：序列化开销微秒级，普通传递就够，Sendable 的约束（布局锁定、共享模块）反而添麻烦；
- **100KB 以上**：序列化毫秒级起步，引用传递的收益明确，上 Sendable；
- **需要带方法/双向共享**：Sendable 是标准答案；
- **一次性大批原始数据（像素、音频采样）**：优先考虑 SharedArrayBuffer 或 Transferable（转移所有权、零拷贝）这类更轻的通道，不一定要建模成 Sendable 类；
- **共享后读写极频繁**：评估锁竞争。有时"每个线程持有一份拷贝、定期合并"比实时共享更快。

## 排查与验证

- **序列化报错**：`taskpool: failed to serialize arguments` 过滤 ArkCompiler Error 日志，定位到具体参数类型；
- **instanceof 失效**：检查定义类的文件顶部有没有 `"use shared"`；
- **数据不一致/偶发崩溃**：共享对象有没有未加锁的并发写；布局有没有被偷偷改（增删属性会在运行时报错）；
- **验证性能收益**：改造前后各跑一次 CPU 会话，对比传输段耗时。官方的定位是"解决 100KB 以上大数据与带方法对象两个关键场景"，不要指望它优化小对象路径。

## 版本演进

- API 11：Sendable 体系首批能力（@Sendable、ISendable、collections Sendable 容器）；
- API 12（5.0）：AsyncLock 加入 @kit.ArkTS；Sendable 在 TaskPool/Worker 全场景贯通；
- API 18+：sendableContextManager 开放，Context 跨线程标准化；sendablePreferences（共享用户首选项）等系统 API 开始基于 Sendable 提供多线程安全访问；序列化工具（类 JSON 的 Sendable 序列化/反序列化）补进 ArkTS 工具集。

## 参考资料

- 华为开发者文档：《Sendable 协议》（arkts-sendable：默认引用传递、支持拷贝传递）
- 华为开发者文档：《Sendable 使用指南》（sendable-guide：100KB 约 1ms、两大关键场景）
- 华为开发者文档：《Sendable 约束》（sendable-constraints：布局不可变等规则）
- 华为开发者文档：《Native 线程间共享》（native-interthread-shared：引用传递无拷贝、性能对比结论）
- 华为开发者文档：《并发 FAQ》（concurrency-faq：instanceof 与 "use shared" 共享模块）
- 华为开发者文档：API 参考 `@ohos.app.ability.sendableContextManager`（convertFromContext、setEventHubMultithreadingEnabled、postMessageWithSharedSendable）
- 华为开发者文档：《ArkTS 术语表》（arkts-glossary：Sendable 定义——避免结构化克隆的深拷贝开销）
- OpenHarmony 仓库 `openharmony/arkcompiler_ets_runtime`（Sendable 共享堆与 AsyncLock 实现）
