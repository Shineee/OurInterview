# 引用类型与值


Java 虚拟机中有三种引用类型：类类型（ Class Types）、数组类型（ Array Types）和接口类型（ Interface Types）。这些引用类型的值分别由类实例、数组实例和实现了某个接口的类实例或数组实例动态创建。其中，数组类型还包含一个单一维度（即长度不由其类型决定）的组件类型（ Component Type），一个数组的组件类型也可以是数组。但从任意一个数组开始，如果发现其组件类型也是数组类型的话，继续重复取这个数组的组件类型，这样操作不断执行，最终一定可以遇到组件类型不是数组的情况，这时就把这种类型成为数组类型的元素类型（ Element Type）。数组的元素类型必须是原始类型、类类型或者接口类型之中的一种。在引用类型的值中还有一个特殊的值： null，当一个引用不指向任何对象的时候，它的值就用 null 来表示。一个为 null 的引用，在没有上下文的情况下不具备任何实际的类型，但是有具体上下文时它可转型为任意的引用类型。引用类型的默认值就是 null。Java 虚拟机规范并没有规定 null 在虚拟机实现中应当怎样编码表示。 

























