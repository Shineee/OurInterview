# Java 常用框架概念索引

面试中除了语言本身，框架的核心机制也是高频考点。本页汇总 Java 后端（Spring 生态）与 Android 侧常见框架的关键概念，只讲原理层面的"面试必知"，具体用法以官方文档为准。

## Spring 核心概念

Spring 的两个基石是 IoC（控制反转）与 AOP（面向切面编程），其余特性大多围绕这两点展开。

### IoC（控制反转）与 DI（依赖注入）

- 传统写法中对象自己 `new` 依赖；IoC 把"创建与装配对象"的权力交给 Spring 容器。
- 常见注入方式：构造器注入（推荐）、setter 注入、字段注入（`@Autowired`）。
- 容器读取配置/注解 → 解析 bean 定义（BeanDefinition）→ 实例化、填充属性、Aware 回调 → 初始化（`@PostConstruct`、`InitializingBean`、`init-method`）→ 放入单例池 → 销毁（`@PreDestroy`、`destroy-method`）。

**面试追问：Spring Bean 的生命周期？**

简版答案：实例化（构造器）→ 属性填充 → BeanName/BeanFactory/Aware 接口回调 → `BeanPostProcessor#postProcessBeforeInitialization` → 初始化方法（`@PostConstruct` → `afterPropertiesSet` → `init-method`）→ `postProcessAfterInitialization`（AOP 代理在此生成）→ 使用 → 销毁回调。单例由容器管理生命周期，原型作用域的 Bean 交给调用方管理，容器不负责销毁。

**面试追问：循环依赖如何解决？**

Spring 通过三级缓存解决单例字段/setter 注入的循环依赖：一级缓存放完整 Bean，二级缓存放早期暴露的原始对象（可被代理包装），三级缓存放 ObjectFactory 以支持 AOP 代理的提前暴露。构造器注入的循环依赖无法自动解决，需要 `@Lazy` 或 `@DependsOn` 等手段。

**面试追问：`@Autowired` 与 `@Resource` 的区别？**

- `@Autowired` 是 Spring 的注解，默认按类型注入，可配合 `@Qualifier` 按名称限定。
- `@Resource` 是 JSR-250 注解，默认按名称注入，找不到再按类型。

### AOP（面向切面编程）

- 把日志、事务、权限等横切逻辑从业务代码中剥离。
- 核心术语：切点（Pointcut）、通知（Advice：Before/After/Around/AfterReturning/AfterThrowing）、切面（Aspect）。
- 实现方式：动态代理。目标类实现了接口时默认用 JDK 动态代理（基于接口）；未实现接口时用 CGLIB（基于子类字节码生成）。Spring Boot 2.x 起默认倾向于 CGLIB。

**面试追问：Spring 事务失效的常见场景？**

方法非 public、同类内部调用（代理对象未介入）、异常被 catch 吞掉、抛出的是受检异常而未配置 `rollbackFor`、数据库引擎不支持回滚、多线程调用不在同一事务上下文等。

### Spring MVC 请求处理流程（一句话版）

`DispatcherServlet` 接收请求 → `HandlerMapping` 找到处理器 → `HandlerAdapter` 执行（前置拦截器）→ 返回 `ModelAndView` → `ViewResolver` 解析视图 → 渲染响应；前后端分离场景下常由 `@ResponseBody` 直接序列化 JSON 返回。

### Spring Boot 与 Spring Cloud（概念级）

- Spring Boot：约定优于配置，自动装配（AutoConfiguration 按条件加载 Bean）、起步依赖、内嵌容器。
- Spring Cloud：微服务配套——服务注册发现（Nacos/Eureka）、配置中心、网关、熔断限流（Resilience4j/Sentinel）、链路追踪。

## Android 相关框架（简述）

- **Jetpack（AndroidX）**：架构组件（ViewModel/LiveData/Room/DataStore）、导航、Compose 等，核心思想是把生命周期感知、持久化、UI 状态管理标准化。
- **Retrofit**：注解 + 动态代理把接口声明转换为 HTTP 请求，配合 OkHttp 执行；转换器（Converter）负责序列化。
- **OkHttp**：拦截器链（Interceptor Chain）是其核心设计，连接池复用、缓存、重试都在拦截器层完成。
- **Gson/Moshi/Jackson**：JSON 反序列化；注意字段名映射、循环引用与泛型擦除导致的 `TypeToken` 用法。
- **Hilt/Dagger**：编译期生成依赖注入代码，服务于 Android 的组件作用域（Activity/ViewModel scope）。
- **协程（kotlinx.coroutines）**：不是库而是语言级并发模型的延伸，`suspend` 函数、结构化并发、`Flow` 响应式流。

## 复习建议

框架面试题万变不离其宗：**Bean 生命周期、代理方式、事务传播、请求分发、线程/协程模型**。把这些原理讲清楚，比背 API 更能体现"学精学通"。
