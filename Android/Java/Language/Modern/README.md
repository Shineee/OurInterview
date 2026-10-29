# Java 8+ 现代语言特性

Java 8 是 Java 近十年最重要的一次版本升级：函数式编程入门（Lambda、方法引用、函数式接口）、Stream 流式处理、Optional 空值治理、java.time 新日期 API，以及并发上的 CompletableFuture。本目录按主题深挖机制与正确用法：

- [Lambda](Lambda.md) —— 语法、函数式接口、变量捕获、与匿名类的区别、invokedynamic 实现
- [FunctionalInterface](FunctionalInterface.md) —— Predicate/Function/Consumer/Supplier/UnaryOperator 及常用默认方法
- [Stream](Stream.md) —— 创建、惰性求值与短路、map/filter/reduce/collect、并行流的代价
- [Optional](Optional.md) —— 正确用法与反模式（orElse vs orElseGet、不作为字段）
- [DateAndTime](DateAndTime.md) —— java.time 包、与旧 Date/SimpleDateFormat 对比
- [ConcurrencyModern](ConcurrencyModern.md) —— CompletableFuture 组合、StampedLock、虚拟线程（Project Loom）面试答法

阅读顺序建议：先 FunctionalInterface → Lambda → Stream → Optional；DateAndTime 与 ConcurrencyModern 可独立阅读。
