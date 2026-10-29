# Java 二分查找

二分查找（Binary Search）在**有序数组**上以每次排除一半的节奏定位目标，平均/最坏均为 **O(log n)**。代码只有十几行，但**边界写错是重灾区**——面试手写必须一次通过。

## 基本思想

- 前提：序列有序（通常升序）。
- 维护闭区间 `[lo, hi]`，取中点 `mid`：
  - `a[mid] == target` → 找到；
  - `a[mid] < target` → 目标在右半，`lo = mid + 1`；
  - `a[mid] > target` → 目标在左半，`hi = mid - 1`。
- `lo > hi` 时区间为空 → 未找到，返回 -1。

## 标准模板（闭区间，推荐背这一个）

```java
public class BinarySearch {

    /** 有序数组中查找 target，返回下标；不存在返回 -1 */
    public static int search(int[] a, int target) {
        int lo = 0, hi = a.length - 1;         // 闭区间 [lo, hi]
        while (lo <= hi) {                     // 区间为空时退出
            int mid = lo + ((hi - lo) >> 1);   // 防溢出写法，勿用 (lo+hi)/2
            if (a[mid] == target) {
                return mid;
            } else if (a[mid] < target) {
                lo = mid + 1;
            } else {
                hi = mid - 1;
            }
        }
        return -1;
    }

    public static void main(String[] args) {
        int[] a = {1, 3, 5, 7, 9, 11, 13};
        System.out.println(search(a, 7));      // 3
        System.out.println(search(a, 6));      // -1
    }
}
```

三条边界铁律（对应最常考的三个追问）：

1. **循环条件用 `lo <= hi`**（闭区间）——若误写 `<`，两元素相等时区间会剩不下而漏查。
2. **移动用 `mid + 1` / `mid - 1`**——若误写 `mid`，单元素区间会死循环。
3. **中点用 `lo + ((hi - lo) >> 1)`**——`(lo + hi) / 2` 在 `lo + hi` 超过 `Integer.MAX_VALUE` 时溢出为负数（Java 无无符号右移问题但除法会得到负下标）；也可写 `lo + (hi - lo) / 2`。

## 变体：左边界（第一个 ≥ target 的位置）

很多实际问题不是"找到任意一个"，而是**插入位置 / 第一个等于 target 的下标 / lower_bound**：

```java
/** 返回第一个 >= target 的下标（lower_bound）；不存在返回 a.length */
public static int lowerBound(int[] a, int target) {
    int lo = 0, hi = a.length;                 // 注意：半开区间 [lo, hi)
    while (lo < hi) {
        int mid = lo + ((hi - lo) >> 1);
        if (a[mid] < target) {
            lo = mid + 1;
        } else {
            hi = mid;                          // a[mid] >= target，收缩右界但保留 mid
        }
    }
    return lo;                                 // 即 target 应插入的位置
}

// 用法
// int pos = lowerBound(a, target);            // 插入点
// boolean found = pos < a.length && a[pos] == target;
```

半开区间写法口诀：

- 循环 `lo < hi`（`hi` 取不到）；
- 比较失败时 `hi = mid`（**不减 1**，因为 mid 可能就是答案）；
- 返回 `lo` 即第一个满足条件的位置。

对称地，"最后一个 ≤ target"（upper_bound - 1）把条件反向即可。**掌握一个模板 + 会推导插入位置变体，比背多个散装模板可靠。**

## 复杂度

| 情况 | 时间 | 说明 |
| ---- | ---- | ---- |
| 查找 | O(log n) | 每次砍半，最坏比较次数 ≈ log₂n + 1 |
| 有序表插入（配合移动） | O(log n) 查 + O(n) 移 | 查找省不了搬移 |
| 空间 | O(1) 迭代 | 递归写法 O(log n) 栈 |

10 亿个元素最多约 30 次比较即可定位——这是"对数级威力"的直观例子。

## 应用与延伸

- **标准库对照**：`Arrays.binarySearch(int[], key)`（要求已排序；未找到返回 `-(插入点) - 1`）；`Collections.binarySearch`；`List.indexOf` 不适用（无序假设下只能线性）。
- **`RandomAccess` 与查找**：`Collections.binarySearch` 对实现 `RandomAccess` 的 List（ArrayList）走二分，对 LinkedList 自动退化为顺序遍历——标记接口影响算法选择。
- **旋转有序数组**（LeetCode 33）：二分时判断哪半有序，在有序半内判断 target 是否落在其中。
- **答案二分（binary search on answer）**：单调性问题（如"最小的最大值"）把"检查某个解是否可行"当 `check` 函数对值域二分——面试算法题的通用套路。
- **浮点二分**：循环条件改为 `hi - lo > eps`，注意精度与迭代次数。
- 链表不能随机访问，二分无效（每次取 mid 就 O(n)），应改用其他方法。

## 【面试追问链】

1. **为什么 `mid` 不能写 `(lo + hi) / 2`？** → `lo + hi` 可能溢出 int（两约 15 亿相加超 21 亿），得到负数下标抛 AIOOBE；`lo + (hi - lo) / 2` 或位移写法既防溢出又等价。
2. **`lo <= hi` 与 `lo < hi` 差在哪？** → 闭区间必须 `<=`（`lo == hi` 时还有一个元素没查）；半开区间 `[lo, hi)` 用 `<`（`lo == hi` 表示空）。混用是死循环/漏查的第一大原因。
3. **为什么移动是 `mid ± 1` 而不是 `mid`？** → `mid` 已经判定不等于 target，留在区间里会反复取到它导致不收敛；闭区间下必须排除 mid。
4. **有序数组里有重复元素，怎么找第一个/最后一个 target？** → 二分定位任意一个后向两侧扩散是 O(n)；正确做法是分别用 lower_bound 查 `target` 与 `target + 1` 的插入点，区间即全部相等段——O(log n)。
5. **二分一定比线性快吗？** → 无序数据不能二分；很小的数组（如 ≤ 8）线性更快（分支预测 + 无计算开销）；链表上二分退化。二分的前提是"随机访问 + 单调判定"，缺一不可。
6. **`Arrays.binarySearch` 找不到时返回什么？** → `-(插入点) - 1`（负数），既表示未找到又编码了应插入的位置——为什么不是 -1？因为插入位置本身可能是 0，用 `-insert - 1` 无歧义编码两个信息。
