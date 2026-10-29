# Android 高阶专题（Senior / 架构师向）

> 定位：面向资深 / 专家 / 架构师 / 技术委员面试的深度内容。不追求"背完能过初级面试"，追求**机制级理解 + 工程决策能力 + 可落地的排查路径**。每题按「问题 → 机制级答案 → 为什么这样设计 → 面试官视角 → 追问链」组织。

## 目录

### [系统设计（SystemDesign）](SystemDesign)

- 第1页：设计一个图片加载库（Glide/Coil 级深度）；设计跨进程 IM 消息通道（连接进程 / 心跳 / 有序投递）
- 第2页：设计崩溃 + ANR 监控 SDK（Java/Native/生死簿式采集、采样与聚合上报）；设计组件化路由框架（注册发现、拦截器、降级）
- 第3页：设计题通用框架：需求边界、容量估算、稳定性设计、演进路线、面试官视角汇总

### [稳定性（Stability）](Stability)

- 第1页：崩溃治理全链路——Java 异常、Native crash、ANR 的采集原理与归因方法；生死簿（生命周期日志）在崩溃分析中的用法
- 第2页：ANR 归因实战（Binder、锁、IO、主线程消息队列）；线上质量指标体系（崩溃率/ANR率/卡顿率口径）与灰度放量、熔断策略

### [性能优化（Performance）](Performance)

- 第1页：启动优化体系——测量口径、主线程任务 DAG 编排、类预加载与 OpenJDK Class Preloading、so 预加载与启动任务框架设计
- 第2页：内存（Ashmem/堆外内存/泄漏监控与 dump 分析）、流畅度（Choreographer/掉帧/合成层）、包体积（资源/DEX/SO/混淆）三条线

### [架构演进（ArchitectureEvolution）](ArchitectureEvolution)

- 第1页：单模块 → 多模块 → 多进程的演进路径；接口暴露与 SPI/组件依赖治理；多团队协作下的代码权限与 Review 机制
- 第2页：技术债务管理——度量、分类、还款节奏与组织保障

### [跨端与新技术（CrossSkill）](CrossSkill)

- 第1页：Compose 渲染原理（Slot Table、重组、与 View 体系的差异）；Kotlin 多平台（KMP）概念与面试答法
- 第2页：Flutter 引擎与三棵树架构、小程序对比选型的一句话深度；插件化/热修复原理与合规边界

## 使用建议

- 一级：先能独立把每题的「机制级答案」讲 3 分钟；二级：能画出模块图并解释每个关键决策的 trade-off；三级：能应对面试官沿「追问链」往下钻三层。
- 与 `TechnicalInterview`（体系化基础面试）、`TechnicalWrittenExamination`（笔试题）互补：先基础后高阶。
