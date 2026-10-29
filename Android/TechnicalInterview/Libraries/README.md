# Android Libraries 面试题

* OkHttp 的拦截器责任链机制（源码级）

  > 请求发出后经过 **RealCall.getInterceptorChain**：应用拦截器（addInterceptor）→ 重试/重定向拦截器 → BridgeInterceptor（补 header、gzip、cookie）→ CacheInterceptor（缓存命中则短路）→ ConnectInterceptor（建立/复用连接）→ 网络拦截器（addNetworkInterceptor）→ CallServerInterceptor（真正 IO 读写）。
  >
  > **设计动机**：责任链让横切逻辑（缓存、压缩、证书、日志、重试）解耦；应用拦截器看到完整逻辑请求（含重定向前后），网络拦截器看到真实上行流量。
  >
  > 面试常追问：为什么日志拦截器放应用层还是网络层？——应用层看最终意图，网络层看真实字节（含重写后 header）。

* 连接池（ConnectionPool）与连接复用原理

  > OkHttp 默认 5 分钟空闲连接保留在连接池；同 host+port+协议可复用已建立的 TCP/TLS 连接，避免握手开销（TLS 握手数十到上百 ms）。
  >
  > 实现：RealConnection + ConnectionPool 内部清理线程按 keepAlive 扫；StreamAllocation/分片（Http2 多路复用一条连接跑多请求）。
  >
  > 面试追问 Http2：一条 TCP 连接多路复用（stream），header 压缩 HPACK；Http1.1 则一连接一请求（keep-alive 但串行）。OkHttp 对两者透明管理。

* Retrofit 如何把注解接口变成网络请求（动态代理）

  > 接口方法上的 @GET/@POST/@Body 等注解在**运行时**由 Retrofit.create 通过 **JDK 动态代理（Proxy.newProxyInstance）** 生成实现：invoke 拦截方法调用 → ServiceMethod 解析注解与参数（buildParameterHandler）→ 组装 OkHttp Request → CallAdapter 把 OkHttpCall 包装成 Call/Response/Flow 等类型 → Converter（Gson/Moshi）序列化 body、反序列化响应。
  >
  > **扩展点**：
  > - **CallAdapter**：适配返回类型（RxJava Observable、Coroutine suspend、LiveData）。
  > - **Converter**：body 转换器（GsonConverterFactory 等）。
  >
  > 面试追问："为什么 suspend 函数 Retrofit 直接支持？"——内置 Suspend 接口的 CallAdapter，挂起恢复走 OkHttp 异步回调。

* Retrofit + OkHttp 的线程与异步模型

  > enqueue 是 OkHttp 的异步（Dispatcher 线程池执行），回调默认主线程（Platform.Android 配 MainThreadExecutor）；execute 同步阻塞，不可主线程调用。
  >
  > Dispatcher：限制并发请求（默认 maxRequests=64，perHost=5），空闲清理；这就是"OkHttp 也帮你管线程池"——与自建线程池叠加要防超限。

* Glide 的缓存层级与生命周期绑定（对比 Coil）

  > **Glide 缓存**：活跃资源（弱引用）→ 内存 LruCache（资源）→ 磁盘（原始 DATA / 转码 RESOURCE / 重用 BITMAP）；键含尺寸、变换、签名。
  >
  > **生命周期绑定**：Glide.with(activity/fragment) 创建无 UI RequestManagerFragment 挂到 LifecycleOwner，ON_DESTROY 自动取消请求——避免 Activity 销毁后回调泄漏，是 Glide 相对 ImageLoader 的核心设计。
  >
  > **Coil**：Kotlin 协程实现，更轻、默认 OkHttp 下载、对 Compose 友好（rememberAsyncImagePainter）；缓存策略类似。选型：大项目复杂变换用 Glide 生态稳；Kotlin/Compose 新项目 Coil 更简洁。
  >
  > 追问：Bitmap 复用（inBitmap）与硬件位图（HARDWARE config）——大图显示避免抖动，Coil/Glide 都支持硬件位图（API 26+）减少内存拷贝。

* 面试官视角

  > **考察点**：网络栈是否读过拦截器顺序、连接池与 Http2；Retrofit 是否理解动态代理本质与扩展点。
  >
  > **加分**：责任链顺序背得出、ConnectionPool keepAlive、CallAdapter/Converter 区别、Glide 无 UI Fragment 绑定生命周期。
  >
  > **减分**：把 Retrofit 当"注解自动发请求"无代理概念；不知道连接复用条件；图片库只说"有 LruCache"。

---

* Gson 与 Moshi 的差异与选型（含序列化陷阱）

  > **Gson**：反射为主（字段名适配 TypeAdapter），API 简单、生态大；问题：反射开销、ProGuard 混淆需 keep 规则、Kotlin 默认参数/非空性支持弱（需 TypeAdapter 或 kotlin adapter）。
  >
  > **Moshi**：官方 Kotlin 友好，优先 codegen（@JsonClass(generateAdapter=true)）生成适配器，无反射、混淆友好、性能更好；对 sealed class/polymorphism 支持更清晰。
  >
  > **陷阱**：Gson 反序列化 Kotlin data class 时若无无参构造/默认值，可能给非空字段赋 null（反射绕过构造器校验）——线上"字段有值却 NPE"经典坑；Moshi codegen 走构造器更安全。
  >
  > 选型：新 Kotlin 项目 Moshi（或 kotlinx.serialization）；存量 Gson 注意 keep 与 Kotlin 支持。

* Room 的编译期机制与线程/迁移要点（链接 Architecture，这里补库细节）

  > Room 在编译期校验 SQL 与实体映射，生成 DAO 实现（非反射）；@Entity 定义表、@Database 持有、@Dao 持查询；@Relation 支持嵌套查询。
  >
  > **线程**：默认禁止主线程查询；Kotlin 挂起 DAO 自动 IO。**事务**：@Transaction 方法或 runInTransaction；多表一致更新用它。
  >
  > **迁移**：版本升级必须 Migration 或 fallbackToDestructiveMigration（后者丢数据，仅适合可重建数据）；线上忘写 Migration 是崩溃高发。
  >
  > 面试追问：Room 与 Flow 结合——@Query 返回 Flow 会在表变更时自动重发（InvalidationTracker 靠触发器/InvalidationTracker 观察）。

* DataBinding 与 ViewBinding 的区别与迁移

  > **DataBinding**：布局内 <data> + 表达式绑定，支持双向（@={})、可观察对象（ObservableField/LiveData）；编译期生成绑定类。问题：布局复杂、表达式调试难、增量编译慢。
  >
  > **ViewBinding**：只为每个布局生成类型安全的 findViewById 替代类，无表达式、轻量。推荐简单绑定用 ViewBinding。
  >
  > 迁移：新模块 ViewBinding；存量 DataBinding 按需逐步换；两者可在同一工程共存（不同模块）。
  >
  > 面试追问：ViewBinding 与 findViewById 差异——编译期确定 id、空安全、无反射，本质是编译期代码生成。

* 三方库集成的通用陷阱与治理

  > 1. **版本冲突**：传递依赖不一致 → Gradle dependencyInsight 排查、约束统一版本。
  > 2. **初始化时机**：SDK 在 ContentProvider/Application 静默 init 拖慢启动（链接启动优化）。
  > 3. **线程模型**：库自带线程池与主线程回调约定要遵守，勿在库回调里再阻塞。
  > 4. **混淆规则**：序列化/反射库必须 keep，否则 release 崩（debug 好的 release 崩）。
  > 5. **生命周期**：带 Activity/Context 的库要在 destroy 释放（WebView、图片库除外已绑定）。
  >
  > 治理：统一依赖版本目录（version catalog）、ABI 签名校验第三方 AAR、集成评审"是否必须主线程 init"。

* 面试官视角

  > **考察点**：序列化/数据库/绑定库是否理解生成机制与线上陷阱，而非只会 API。
  >
  > **加分**：Gson 反射给 Kotlin 非空赋 null、Room InvalidationTracker、DataBinding 双向 vs ViewBinding 轻量、版本冲突排查命令。
  >
  > **减分**：推荐 Gson 做高性能 Kotlin 新项目；Room 主线程查询无所谓；不知道混淆 keep。
