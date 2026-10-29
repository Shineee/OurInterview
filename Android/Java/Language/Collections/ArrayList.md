# Java ArrayList

ArrayList 是最常用的 List 实现，底层是一块**可动态扩容的 Object 数组**。理解它的扩容、复杂度与迭代器行为，是集合面试的第一课。

## 底层结构与扩容

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable {

    transient Object[] elementData;
    private int size;
}
```

- 默认构造时并不分配数组（JDK 8 中 `elementData` 指向共享空数组），第一次 `add` 时才分配默认容量 **10**。
- 元素存放在 `elementData`，`size` 记录实际元素个数（数组长度 ≥ size）。
- 添加元素时若 `size == elementData.length`，触发扩容：新容量 = 旧容量 + 旧容量 >> 1，即 **1.5 倍**（`newCapacity = oldCapacity + (oldCapacity >> 1)`）。
- 扩容通过 `Arrays.copyOf` 复制到新数组，**代价是 O(n)**，所以频繁尾部追加的场景建议用 `new ArrayList<>(expectedSize)` 指定初始容量，避免反复扩容。

**面试追问：为什么扩容是 1.5 倍而不是 2 倍？**

1.5 倍在"扩容浪费"与"扩容频率"之间折中：2 倍浪费更少但每次扩容拷贝更多数据、更容易让内存出现无法利用的空洞；1.5 倍留出的旧数组在 GC 前仍可被 `subList` 等视图短暂引用的空间也更可控。另外容量增长用位运算 `>> 1` 实现，非常廉价。

**面试追问：ArrayList 默认容量是多少？扩容后呢？**

首次添加前容量为 0（共享空数组），首次添加分配 10，之后 10 → 15 → 22 → 33 …（1.5 倍向下取整）。用 `new ArrayList<>(n)` 可以一次性分配，`ensureCapacity(minCapacity)` 可在批量添加前手动扩容。

## 增删查复杂度

| 操作 | 平均复杂度 | 说明 |
| ---- | ---- | ---- |
| `get(index)` | O(1) | 数组随机访问 |
| `set(index, e)` | O(1) | 覆盖写 |
| `add(e)` 尾部 | O(1) 摊还 | 偶发扩容 O(n) |
| `add(index, e)` 中间 | O(n) | 后移元素 |
| `remove(index)` | O(n) | 前移元素 |
| `indexOf(o)` | O(n) | 线性查找 |
| `contains(o)` | O(n) | 依赖 `equals` |

一句话结论：**读多写少、随机访问多选 ArrayList；频繁在头部/中间插入删除选 LinkedList**（实际工程中 ArrayList 的 `O(n)` 拷贝往往因缓存局部性反而更快，见 [ArrayListVsLinkedList](../DataStructure/ArrayListVsLinkedList.md)）。

## fail-fast 与 modCount

```java
List<String> list = new ArrayList<>(Arrays.asList("a", "b", "c"));
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    if ("b".equals(s)) {
        list.remove(s);      // 结构性修改，modCount 变化
    }
}
```

- 每次结构性修改（add/remove/clear 等）都会使 `modCount` 自增。
- 迭代器创建时记录 `expectedModCount = modCount`；`next()` 时若两者不等，抛出 `ConcurrentModificationException`——这就是 **fail-fast（快速失败）** 机制。
- 它**不保证**在并发下一定抛异常（只是尽力检测），也不是线程安全机制。
- 正确删除方式：`it.remove()`（迭代器自身的 remove 会同步 `expectedModCount`），或 JDK 8+ 的 `list.removeIf(...)`。

**面试追问：为什么不直接用线程安全的类来避免 CME？**

fail-fast 只是检测手段，不是同步手段。Java 的取向是"单线程下不为并发买单"：需要并发时用 `CopyOnWriteArrayList`（适合读多写少、元素少）或外部加锁，而不是让每次迭代都承担锁开销。

## toArray 与 subList 陷阱

### toArray

```java
String[] arr1 = list.toArray();                 // 返回 Object[]，转型可能失败
String[] arr2 = list.toArray(new String[0]);    // 推荐写法：传空数组，JDK 会按需分配正确类型
String[] arr3 = list.toArray(new String[list.size()]); // 旧写法，可能浪费或二次分配
```

- 直接 `toArray()` 返回 `Object[]`，对 `String list` 做 `String[] a = list.toArray()` 会 `ClassCastException`。
- 推荐 `toArray(new T[0])`：JDK 内部会创建恰好长度的新数组（或复用传入数组），语义清晰且不易出错。

### subList 陷阱

```java
List<Integer> sub = list.subList(0, 3);   // 返回的是原列表的视图，不是副本
sub.clear();                               // 会反映到原 list！
```

- `subList` 返回的只是原列表的一段**视图（view）**，共享同一份底层数组，任何一方的结构性修改都会互相影响。
- 对视图做结构性修改会改变原 list 的 `modCount`，导致**遍历原 list 时抛 CME**。
- 持有 subList 期间原 list 被扩容/重排（如外部 `add` 触发数组替换），视图可能失效甚至抛 `ConcurrentModificationException`——典型 Bug 是把 subList 存为字段后异步使用。
- 需要独立副本时用 `new ArrayList<>(list.subList(from, to))`。

## 与 Vector 的区别

| 维度 | ArrayList | Vector |
| ---- | ---- | ---- |
| 线程安全 | 否 | 方法级 `synchronized` |
| 扩容 | 1.5 倍 | 默认 2 倍（容量增量可指定） | 
| 性能 | 高（无锁开销） | 每个操作都有锁开销 |
| 迭代器 | fail-fast | 迭代器不基于 modCount，改用 `count` 字段（同样不推荐多线程直接用） |

Vector 是 JDK 1.0 遗留类，现代代码几乎不用；需要线程安全列表用 `CopyOnWriteArrayList` 或在应用层加锁。

## 常见面试追问速记

- **`Arrays.asList` 与 `ArrayList` 的区别**：前者定长（`add/remove` 抛 `UnsupportedOperationException`），底层数组直接暴露，可 `set`；且基本类型数组会整包成元素（`int[]` 变成一个元素）。
- **`ArrayList` 与 `LinkedList` 谁快**：随机访问 ArrayList O(1) 锁定胜局；中间插入理论 LinkedList 更优，但缓存不友好 + 对象开销，实测常常 ArrayList 更快。
- **为什么线程不安全还用它**：绝大多数场景是单线程或只读共享；并发是少数，交由上层设计解决。

## 【面试追问链】

1. **为什么扩容是 1.5 倍而不是 2 倍？** → 1.5 倍在扩容频率与内存浪费间折中；2 倍虽拷贝次数少，但每次扩容浪费的空间按指数放大，且更易留下无法被后续分配利用的堆空洞。位运算 `>> 1` 使代价近乎为零。
2. **fail-fast 一定会抛 CME 吗？** → 不一定。它只是"尽力检测"：modCount 比较存在竞态窗口，单线程下结构性修改必抛，多线程下可能漏检也可能误检——所以它不是线程安全机制，只是快速暴露 Bug 的手段。
3. **`modCount` 在 `set(index, e)` 时变吗？** → 不变。`set` 只覆盖元素不改结构，不算结构性修改；只有 add/remove/clear 等改变 `size` 的操作才自增 `modCount`。
4. **`ArrayList` 的 `iterator` 与 `listIterator` 差在哪？** → `listIterator` 支持双向遍历、`set/add/remove` 当前元素、从任意下标起步；`iterator` 只能单向 `next/remove`。两者都基于 `expectedModCount` 做 fail-fast 检查。
5. **`Arrays.asList` 返回的对象能扩容吗？** → 不能。它是 `Arrays$ArrayList`（定长），`add/remove` 抛 `UnsupportedOperationException`，但 `set` 可用；需要可变副本用 `new ArrayList<>(Arrays.asList(...))`。
6. **`subList` 持有期间原 list 扩容了会怎样？** → 视图共享原底层数组引用；原 list 扩容会把 `elementData` 换成新数组，视图仍指向旧数组——数据"看起来还在"但与原 list 脱节，后续通过原 list 的结构修改还会让视图抛 CME。结论：视图生命周期不要长于原 list 的稳定期。
