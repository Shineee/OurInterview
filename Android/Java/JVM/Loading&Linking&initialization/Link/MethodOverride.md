# 方法覆盖

要声明在 C 类中的实例方法 m1 覆盖（ Overriding）另外一个声明在 A 类里的实例方法 m2，当且仅当下面的条件都成立才合法：

* C 是 A 的子类。
* m2 与 m1 拥有相同的名称及方法描述符。
* 下面其中一条成立：
* m2 的权限标志是 ACC_PUBLIC 或 ACC_PROTECTED 或默认权限（既不是ACC_PUBLIC、也不是 ACC_PROTECTED 更不是 ACC_PRIVATE，且和 C 类处于同一运行包下面）。
* m1 覆盖方法 m3， m3 与 m1 不同， m3 也与 m2 不同，并且 m3 覆盖了 m2。 

