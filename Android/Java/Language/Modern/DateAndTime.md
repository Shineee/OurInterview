# Java 8 日期与时间（java.time）

`java.time` 是 Java 8 引入的新日期时间 API（JSR-310），设计参考 Joda-Time，**不可变、线程安全、语义清晰**，全面取代了 `Date`/`Calendar`/`SimpleDateFormat` 的旧世界。

## 旧 API 的三大罪状（为什么要换）

| 问题 | 具体表现 |
| ---- | ---- |
| **可变** | `Date`/`Calendar` 的方法会修改自身（`set`、`add`），作参数传递时可能被调用方悄悄改掉 |
| **非线程安全** | `SimpleDateFormat` 内部有 `Calendar` 状态，多线程共享会错乱日期甚至抛异常——这是线上经典事故 |
| **设计混乱** | `Date` 同时表示日期与时间且月份从 0 开始；`Calendar` 与 `Date` 互相冗余；秒/毫秒字段命名含糊；`Date.getTime()` 返回的是时间戳 |

```java
// 旧写法（反例）
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
// 多线程共享 sdf → 可能得到 "1970-01-17" 之类的错乱结果
Date d = new Date();       // 可变
d.setMonth(1);             // 月份从 0 开始：实际是 2 月
```

## 核心类一览

| 类 | 含义 | 是否带时间 |
| ---- | ---- | ---- |
| `LocalDate` | 日期（年月日） | 否 |
| `LocalTime` | 时间（时分秒纳秒） | 否 |
| `LocalDateTime` | 日期+时间（无时区） | 是，**无时区** |
| `Instant` | 时间戳（UTC 纪元秒+纳秒），机器时间 | 是，绝对时刻 |
| `ZonedDateTime` / `OffsetDateTime` | 带时区/偏移的日期时间 | 是 |
| `Duration` / `Period` | 时间量：时长（秒级）/ 日期量（年月日） | — |

**选型关键**：涉及"人的日期"（生日、账单日）用 `LocalDate`；涉及"某一时刻"（服务端日志、跨时区会议）用 `Instant`/`ZonedDateTime`；`LocalDateTime` 无时区——存库/传输时要明确约定时区，避免歧义。

## 常用操作

```java
// 获取
LocalDate today = LocalDate.now();                 // 系统默认时区的今天
LocalDate date = LocalDate.of(2026, 10, 29);       // 月份 1-12，无需 -1
LocalDate parsed = LocalDate.parse("2026-10-29");  // ISO 格式
LocalDateTime dt = LocalDateTime.of(date, LocalTime.of(18, 30));
Instant stamp = Instant.now();                      // 机器时间戳

// 访问字段
int y = today.getYear();
int m = today.getMonthValue();                      // 1-12
int day = today.getDayOfMonth();
DayOfWeek dow = today.getDayOfWeek();               // MONDAY...

// 运算（不可变：返回新对象）
LocalDate nextWeek = today.plusWeeks(1);
LocalDate lastMonth = today.minusMonths(1);
LocalDateTime plusH = dt.plusHours(2);
LocalDate with = today.withDayOfMonth(1);           // 当月 1 号

// 计算差值
long days = ChronoUnit.DAYS.between(LocalDate.of(2020, 1, 1), today);
Duration dur = Duration.between(LocalTime.of(9), LocalTime.of(10, 30));  // PT1H30M

// 格式化与解析
DateTimeFormatter f1 = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
String s = dt.format(f1);
LocalDateTime back = LocalDateTime.parse(s, f1);

// 时间戳转换
Instant i2 = dt.atZone(ZoneId.systemDefault()).toInstant();
LocalDateTime back2 = LocalDateTime.ofInstant(i2, ZoneId.systemDefault());
```

要点：

- **不可变**：`plus/with/withDayOfMonth` 都返回新实例，旧对象不变——天然线程安全，可安全作为字段/静态常量。
- 月份 `of(2026, 10, 29)` 中 10 就是十月（不像 `Date` 的 `setMonth(9)` 才是十月）。
- `Duration` 适合"墙上时钟经过的时间"，跨 DST 时用 `Period`/`ChronoUnit` 或在 `ZonedDateTime` 上运算更稳妥。

## DateTimeFormatter（对比 SimpleDateFormat）

```java
// SimpleDateFormat（反例：共享即事故）
// private static final SimpleDateFormat SDF = new SimpleDateFormat("yyyy-MM-dd"); // 危险

// DateTimeFormatter：不可变、线程安全，可作静态常量共享
private static final DateTimeFormatter FMT =
        DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");

String text = LocalDateTime.now().format(FMT);
LocalDateTime t = LocalDateTime.parse(text, FMT);
```

| 维度 | SimpleDateFormat | DateTimeFormatter |
| ---- | ---- | ---- |
| 线程安全 | **否**（经典事故源） | **是**（不可变） |
| 可作静态常量 | 不行（或每次 new/ThreadLocal） | 可以 |
| 解析失败 | `ParseException`（受检） | `DateTimeParseException`（非受检） |
| 灵活性 | `SimpleDateFormat` 自定义模式 | 同样支持模式字母，另内置 ISO 常量 |

## 与旧 API 互操作（迁移期）

```java
// Date -> 新 API
Date old = new Date();
Instant i = old.toInstant();
LocalDate d = i.atZone(ZoneId.systemDefault()).toLocalDate();

// 新 API -> Date
Date fromNew = Date.from(Instant.now());

// Calendar
Instant i2 = calendar.toInstant();
```

迁移建议：新代码一律 `java.time`；边界上用 `Date.from`/`toInstant` 转换；彻底消灭共享 `SimpleDateFormat`。

## 常见面试追问

- **为什么新 API 不可变？** → 不可变 + 无 setter 才能保证线程安全与作为值对象传递的安全性；代价是运算产生新对象，但日期时间对象创建成本低、生命周期短。
- **`LocalDateTime` 和 `Instant` 怎么选？** → `LocalDateTime` 是"墙上时间"（没有时区的年月日时分秒），跨时区会歧义；`Instant` 是时间轴上的绝对点（UTC），机器间比较/存储用它。服务端存时间建议 UTC（Instant/epoch millis），展示时再转用户时区。
- **时区处理的坑？** → `LocalDate.now()` 用系统默认时区，容器时区（常为 UTC）与业务时区不一致会导致"日期差一天"；涉及用户日期时应显式 `ZoneId.of("Asia/Shanghai")`，不要依赖环境默认。
- **`java.sql.Date/Timestamp` 还有用吗？** → JDBC 旧类型；现代驱动可直接用 `LocalDate`/`Instant`（JDBC 4.2+）。混合项目中在 DAO 边界转换即可。

## 【面试追问链】

1. **`SimpleDateFormat` 线程不安全的根因？** → 它内部解析/格式化都依赖一个可变 `Calendar` 实例作为状态，多线程交错调用会互相污染——修复要么每次 new、要么 ThreadLocal、要么直接换 `DateTimeFormatter`。
2. **`Date` 的月份为什么从 0 开始？** → 历史设计（类 Unix 传统 + 1900 年偏移），是无数 Bug 的源头；`java.time` 修正为 1-12，同时保留 `Month` 枚举可读性。
3. **`Duration.between` 和 `ChronoUnit.DAYS.between` 有区别吗？** → 有。前者返回精确时长（含小时分秒），跨月/月长不一时"天数"语义不同；后者按日期单位截断计算日历日差异。"过了一天"用日历语义还是时长语义要想清楚。
4. **为什么不用时间戳 long 存一切？** → 可以且高效，但丢失可读性与类型语义（是秒还是毫秒？哪个时区的日期？）；`Instant`/`LocalDate` 把语义编码进类型，格式化与日历运算交给库。
5. **新 API 真的零 GC 压力吗？** → 不是——每次 `plus` 都新建对象。热点循环里应复用或改用 `epochDay`/`epochSecond` 做数值运算，或缓存 `DateTimeFormatter`（它贵在初始化，本身线程安全可共享）。
6. **`ZonedDateTime` 与 `OffsetDateTime` 差在哪？** → 带时区的带规则（夏令时、历史时区变化），`Offset` 只有一个固定偏移量。跨夏令时地区用 `ZonedDateTime`；数据库 `TIMESTAMP WITH TIME ZONE` 映射多用 `OffsetDateTime`/`Instant`。
