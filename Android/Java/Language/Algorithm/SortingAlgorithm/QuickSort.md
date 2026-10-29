# Java 快速排序

快速排序是**分治**思想的代表：选一个基准（pivot），把小于它的放左边、大于它的放右边（partition），对左右子区间递归——平均是综合性能最好的比较排序之一，也是 `Arrays.sort(int[])` 对基本类型使用的双轴快排（Dual-Pivot Quicksort）的基础思想。

## 基本思想（partition）

1. 选择基准（常见：取末元素、首元素、三数取中 median-of-three）。
2. 分区：将数组重新排列为 `[小于 pivot | 大于等于 pivot]`，返回 pivot 的最终下标。
3. 对 pivot 左右子数组递归，直到区间长度 ≤ 1。

```
[ 7 | 2 5 1 8 3 ]   pivot = 7?  例：取末元素
partition 后：[ 2 5 1 3 | 7 | 8 ]   返回 7 的下标
递归左区间 [2 5 1 3]、右区间 [8]
```

## 用 java 实现（末元素为 pivot，Lomuto 分区）

```java
public class QuickSort {

    public static void quickSort(int[] a, int low, int high) {
        if (low < high) {
            int p = partition(a, low, high);
            quickSort(a, low, p - 1);        // 左子区间
            quickSort(a, p + 1, high);       // 右子区间
        }
    }

    /** 以 a[high] 为 pivot 重排区间，返回 pivot 归位后的下标 */
    private static int partition(int[] a, int low, int high) {
        int pivot = a[high];
        int i = low - 1;                     // i 指向"小于区"的末尾
        for (int j = low; j < high; j++) {
            if (a[j] < pivot) {
                i++;
                swap(a, i, j);
            }
        }
        swap(a, i + 1, high);                // pivot 归位
        return i + 1;
    }

    private static void swap(int[] a, int x, int y) {
        int t = a[x];
        a[x] = a[y];
        a[y] = t;
    }

    public static void main(String[] args) {
        int[] a = {54, 6, 3, 78, 34, 12, 45, 56, 100};
        quickSort(a, 0, a.length - 1);
        for (int v : a) {
            System.out.print(v + " ");        // 3 6 12 34 45 54 56 78 100
        }
    }
}
```

分区也可以用**双指针从两端向中间**的经典写法（一趟交换次数更少，工程上更常见）；三数取中 + 区间内随机 pivot 可显著降低最坏情况概率。

## 复杂度分析

| 情况 | 时间复杂度 | 说明 |
| ---- | ---- | ---- |
| 平均 | **O(n log n)** | 每层划分 O(n)，共 O(log n) 层 |
| 最好 | O(n log n) | 每次大致对半分 |
| **最坏** | **O(n²)** | 每次划分极不均（如已有序 + 固定取末元素） |
| 空间 | O(log n) | 递归栈平均深度；最坏 O(n) |
| 稳定性 | **不稳定** | 跨区间交换会打乱相等元素相对顺序 |

**最坏情况成因**（面试重点）：

- 输入已基本有序 + 基准取固定端点 → 每次 partition 只切下一个元素，区间退化为 n-1 → 递归 n 层，比较总次数 ~ n²/2。
- 全相等元素 + 朴素分区同样退化（除非分区能均分相等块）。
- 缓解手段：**三数取中**、**随机 pivot**、**小区间切换插入排序**、**递归改尾递归/显式栈**、JDK 的双轴快排（两个 pivot 三路划分，对大量重复元素天然友好）。

**为什么平均是 O(n log n)？** 划分规模期望满足递归式 `T(n) = T(k) + T(n-1-k) + O(n)`，当 k 均匀分布时期望每次对半，层数 O(log n)，每层合计 O(n)。

## 快排 vs 归并 vs 堆排（速记）

| | 快排 | 归并 | 堆排 |
| ---- | ---- | ---- | ---- |
| 平均 | O(n log n) | O(n log n) | O(n log n) |
| 最坏 | O(n²) | O(n log n) | O(n log n) |
| 空间 | O(log n) 栈 | O(n) 辅助数组 | O(1) |
| 稳定 | 否 | **是** | 否 |
| 缓存 | **好**（局部性） | 中 | 差（跳跃访问堆） |
| 实践 | 默认首选 | 稳定外部排序/链表排序 | 内存受限且要求 O(1) |

实际应用：`Arrays.sort(int[])` = 双轴快排变体；`Arrays.sort(Object[])` = TimSort（归并+插入的稳定排序）。

## 【面试追问链】

1. **为什么快排比冒泡/插入快？** → 冒泡每轮只相邻交换、信息利用低；快排通过 partition **一次就确定 pivot 的最终位置**并把区间近乎对半，比较信息被充分利用——分治把"每层 O(n) × O(log n) 层"压过了 O(n²)。
2. **partition 返回什么、递归边界怎么写？** → 返回 pivot 归位下标 p，递归 `[low, p-1]` 与 `[p+1, high]`，终止条件 `low < high`（长度 0/1 天然有序）。漏掉边界会导致栈溢出或死循环。
3. **快排不稳定会带来什么实际后果？** → 对复合键排序时相等键的相对次序丢失（如先按分数再按姓名排序，分数相同的姓名可能乱序）——此时必须用稳定排序（TimSort/归并）或把次级键加入比较器。
4. **如何 O(1) 空间做分区？** → 双指针 Hoare 分区原地交换；但 Hoare 与 Lomuto 的 pivot 归位细节不同，Hoare 不保证 pivot 最终停在返回位置（经典实现返回的分界点两侧都不含 pivot 的固定位置），写代码时要按所选方案的一致语义处理。
5. **递归深度为什么是 O(log n) 而不是 O(n)？** → 平均每次对半 → 深度 log₂n；最坏每次只减 1 → 深度 n，可能栈溢出。随机化 pivot 把最坏情况变成概率极低事件，这是"随机化算法"面试的标准例子。
6. **大量重复元素时快排为什么退化、怎么解决？** → 朴素 `< pivot` 分区把所有等于 pivot 的元素全堆到一侧 → 三路分区（< / == / >，荷兰国旗问题）或双轴快排把相等块整体放中间，恢复对半效果。
