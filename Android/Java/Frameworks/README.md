# Android 相关框架概念

面试中除了语言本身，框架的核心机制也是高频考点。本页只保留 **Android 工程**里会考到的框架原理，不涉及 Java Web / Spring 生态。

## Jetpack（AndroidX）

- 架构组件（ViewModel / LiveData / Room / DataStore）、导航、Compose 等。
- 核心思想：把生命周期感知、持久化、UI 状态管理标准化，避免 Activity/Fragment 承担一切。

## Retrofit

- 注解 + 动态代理把接口声明转换为 HTTP 请求，配合 OkHttp 执行。
- 扩展点：CallAdapter（返回类型适配）、Converter（序列化）。
- 面试常问：为什么用动态代理、注解如何变成 Request、与 RxJava/协程如何对接。

## OkHttp

- 拦截器链（Interceptor Chain）是核心设计：连接池复用、缓存、重试、鉴权都在拦截器层完成。
- 追问方向：责任链模式、HTTP/2 多路复用、连接池清理任务。

## Gson / Moshi / Jackson

- JSON 序列化 / 反序列化。
- 注意：字段名映射、循环引用、泛型擦除下的 `TypeToken`、Kotlin 空安全与默认值。

## Hilt / Dagger

- 编译期生成依赖注入代码，服务于 Android 组件作用域（Activity / ViewModel / Singleton scope）。
- 与「手工 Service Locator」对比：可测试性、作用域边界、编译期校验。

## kotlinx.coroutines

- 不是普通工具库，而是语言级并发模型的延伸：`suspend`、结构化并发、`Flow`。
- 与 Handler / 线程池 / AsyncTask 的对照见 [Kotlin 协程](../../Kotlin/Coroutines)。

## 复习建议

框架面试题万变不离其宗：**生命周期绑定、代理/动态生成、拦截器与责任链、线程/协程模型、依赖注入作用域**。把这些原理讲清楚，比背 API 更能体现「学精学通」。
