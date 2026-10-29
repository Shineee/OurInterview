# Android 类库层笔试题

* OkHttp、Retrofit、Glide、Gson、Room 各解决什么问题？

  > OkHttp：HTTP 客户端，负责连接池、拦截器、请求重试等底层网络通信。
  >
  > Retrofit：在 OkHttp 之上用注解 + 动态代理把接口定义转成 HTTP 请求，简化 API 调用。
  >
  > Glide：图片加载库，负责网络/本地/资源图片加载、缓存与生命周期绑定。
  >
  > Gson：JSON 与 Java 对象互转的序列化库。
  >
  > Room：Jetpack 的 SQLite 抽象层，编译期校验 SQL，返回 LiveData/RxJava/Flow 可观察结果。

* Retrofit 的核心原理用一句话概括，并简要展开。

  > 动态代理 + 注解解析：通过 Proxy 为接口生成实现，调用方法时解析 @GET/@POST 等注解和参数注解，组装成 OkHttp 的 Request 发起请求，再用 Converter 把响应体转成目标类型。
  >
  > 关键点：create() 返回的是 JDK 动态代理对象；invoke 中 ServiceMethod 解析并缓存注解信息；CallAdapter 适配返回类型（Call/Flow/CompletableFuture 等）。

* OkHttp 的拦截器（Interceptor）有什么用？典型应用有哪些？

  > 把一次请求拆成责任链：应用拦截器 -> 重定向/重试 -> 连接池 -> 读写超时 -> 楼梯关拦截器 -> 网络拦截器。
  >
  > 典型应用：统一添加 Token/公共头、请求日志、失败重试、缓存控制、证书校验、Mock 数据。

* Glide 的缓存层级有哪些？

  > 活跃资源（ActiveResources）：正在使用的资源，弱引用持有。
  >
  > 内存缓存（MemoryCache）：LRU 算法，最近未使用的资源。
  >
  > 磁盘缓存分两层：ResourceCache（解码后的数据）与 DataCache（原始未解码数据，如网络字节流）。
  >
  > 加载顺序：活跃资源 -> 内存缓存 -> 磁盘缓存 -> 网络。Glide 还会按 Activity/Fragment 生命周期自动暂停恢复请求。

* Gson 的 TypeToken 是干什么的？

  > 用于在泛型擦除后保留集合等参数化类型信息，如 new TypeToken<List<User>>(){}.getType()，使 Gson 能正确反序列化为 List<User> 而不是 List<LinkedTreeMap>。
  >
  > 原理：匿名内部类可以拿到父类的泛型实际类型参数。

* Room 相比直接用 SQLite 有什么优势？@Dao、@Entity、@Database 各是什么？

  > 优势：编译期校验 SQL（写错列名直接编译报错）、消除大量样板代码、支持编译期生成实现类、可配合 Flow/LiveData 做观察。
  >
  > @Entity：表结构映射；@Dao：数据访问接口（编译期生成实现）；@Database：数据库入口，持有表列表和版本号；@Query/@Insert/@Update/@Delete 标注具体操作。

---

* 为什么不能在子线程更新 UI？（结合 ViewRootImpl 说明）

  > ViewRootImpl.checkThread 会校验当前线程是否等于创建 ViewRootImpl 的线程（通常是主线程），不是则抛 CalledFromWrongThreadException。
  >
  > 根本原因不是"View 不是线程安全的"这么简单——如果只是线程安全，加锁即可；Android 选择在 checkThread 做单线程约束，是为了避免复杂的加锁开销，保证 UI 操作串行。
  >
  > 补充：子线程可以更新 UI 的特例——ViewRootImpl 尚未创建时（如 onCreate 里刚 startActivity 还没走到 handleResumeActivity，或子线程手动 new ViewRootImpl 的理论场景）；实际开发中不要依赖这个特例。

* AsyncTask 使用时有哪些注意点？

  > 它是串行执行的（Android 3.0 后默认 execute 串行，executeOnExecutor 可并行）；onPreExecute/onPostExecute 运行在主线程，doInBackground 运行在传入的线程池。
  >
  > 生命周期风险：AsyncTask 持有 Activity 引用，Activity 销毁后仍回调可能导致泄漏/崩溃；官方在 API 30 已废弃。
  >
  > 配置变化（如转屏）时回调会作用到旧实例；现代替代方案是协程/RxJava/LiveData。

* RxJava 的线程切换原理是什么？

  > 通过 subscribeOn 指定 subscribe（含上游数据产生）运行的线程：内部通过 ObservableCreate 等操作符把"创建任务"这一步包在指定线程执行；observeOn 指定下游 observe 运行的线程：在 ObserveOnObserver 中把事件切到指定 Scheduler 的 Handler post 出去。
  >
  > subscribeOn 只对上游生效一次（多次调用以第一次为准），observeOn 可多次切换下游。

* Kotlin 协程的 suspend 原理一句话概括，并说出其在 Android 中的优势。

  > 编译器把 suspend 函数编译成状态机，通过 CPS 变换和 Continuation 传递控制流，挂起时不阻塞线程，恢复时可能在其他线程继续执行。
  >
  > Android 优势：viewModelScope/lifecycleScope 自动随生命周期取消，避免泄漏；结构化并发；与 Flow 组合方便做响应式 UI。

* Flow 与 LiveData 在 Android 中如何选择？

  > LiveData 生命周期感知、粘性事件问题（历史版本）、仅主线程可 observe；适合简单 UI 状态。
  >
  > Flow 更通用（可在任意作用域收集）、操作符丰富、支持背压（buffer）、可搭配 StateFlow/SharedFlow 做事件总线；需用 repeatOnLifecycle 或 flowWithLifecycle 处理生命周期。
  >
  > 官方趋势：新代码优先 StateFlow/SharedFlow，LiveData 仍可用但不再扩展新特性。

* 列举你用过的网络优化手段（笔试简答）。

  > 连接复用与连接池（OkHttp 默认）、gzip/Brotli 压缩、HTTP 缓存（Cache-Control）、图片压缩与合适格式（WebP）、CDN、请求合并与防抖、弱网超时与重试策略、protobuf 替代 JSON 减小体积、DNS 优化与 HttpDNS。
