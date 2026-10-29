# OurInterview

本项目以另外的角度来看待面试题，避免题海战术，把面试题中的点融汇到技术中，提倡把技术学精、学通，以不变胜万变。

**全书以 Android 工程师求职 / 资深晋升为主线**：Java、Kotlin 不作为独立技术栈叙述，而是 Android 大前提下的语言与运行时地基。高阶内容按「机制级答案 → 设计动机 → 面试官视角 → 追问链」组织，面向资深 / 专家 / 架构师面试深度。

# 目录结构

```
OurInterview/
├── Android/                          # 技术主线
│   ├── TechnicalInterview/           # 技术面试：应用层 / 框架 / 三方库 / 运行时 / 内核
│   ├── TechnicalWrittenExamination/  # 技术笔试：短小精悍的基础与辨析
│   ├── Senior/                       # 高阶：系统设计 / 稳定性 / 性能 / 架构演进 / 跨端
│   ├── Java/                         # Android 工程师向 Java：语言 / JVM / 高阶
│   └── Kotlin/                       # Android 一等公民：语言 / 协程 Flow / 实战 / 高阶
└── HR/                               # 非技术面
```

# 学习路径

| 阶段 | 入口 | 目标 |
|------|------|------|
| 1. 应用层基础 | [技术面试](Android/TechnicalInterview) · [技术笔试](Android/TechnicalWrittenExamination) | Activity/四大组件、异步、View、架构、三方库、Binder |
| 2. 语言地基 | [Kotlin](Android/Kotlin) · [Java](Android/Java) | 协程/Flow 主战场；集合、并发、GC/ART 排查底座 |
| 3. 高阶专题 | [Android Senior](Android/Senior) · [Java Senior](Android/Java/Senior) | 系统设计、稳定性、性能、架构决策与面试叙事 |
| 4. 非技术面 | [HR 面试](HR/Interview) | 考察点鉴别、回答结构、薪资谈判与反问 |

# 内容说明

- **技术面试 / 笔试**：按 Android 五层（应用层、框架层、类库层、运行时、内核）组织，覆盖组件生命周期、Handler/协程、View 绘制与事件、常用库原理、ART 与 Binder。
- **Kotlin**：空安全与语言机制、协程编译产物与取消/异常矩阵、Flow 冷热与背压、Android 实战（viewModelScope、StateFlow）、KMP 与工程治理。
- **Java**：面向 Android 的语言与 JVM 深度（非后端专场）——集合实现细节、并发 AQS、GC 与泄漏排查，均尽量映射到 ART/客户端场景。
- **Senior**：系统设计（图片库/IM 通道/监控 SDK/路由）、崩溃与 ANR 治理、启动/内存/流畅度/包体积、模块化与多进程演进、Compose/KMP/Flutter 选型答法。
- **HR**：自我介绍、优缺点、离职原因、职业规划、薪资结构与谈判、反问建议——写清「面试官在鉴别什么」。

# Interview Introduction

- 种类

  > 面试一般分为技术面试、技术笔试、上机、HR 面试、部门领导面试等

- 过程

  > 通常情况，自行寻找招聘网站、猎头、内推进行投递简历，审核通过后雇主或中间人会电话或邮件通知本人，确定好面试时间，随后进入面试。
  >
  > 第一轮是技术面试，内容多集中在简历中的项目、个人履历与基本信息，轮数根据雇主要求伸缩。
  >
  > 第二轮是技术笔试，可能穿插在技术面中或独立进行，偏算法、数据结构与基础灵活运用。
  >
  > 第三轮是 HR 面试，了解人品与团队文化匹配，以及薪资福利、上下班、地点、发展等。
  >
  > 第四轮是部门领导面试，能到这一轮基本可以发放 offer，可对 HR 面未问清的问题做补充。

- 结果

  > 确定无误后，领导会通知 HR 向你邮箱发送 offer 并电话通知；确认无误需回复邮件接受。未通过时，有节操的公司会第一时间通知；超过一周未回复可考虑别的机会。

# Version

1.3.163.20161128_release

# License

The [MIT](LICENSE) License (MIT)

Copyright (c) 2016 Shine

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.
