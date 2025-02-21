# why rust does not need GC at runtime
所有权和借用规则在编译阶段静态分析内存使用，无需运行时垃圾回收.

* 传统的 GC（如 Java、Go 的 GC）需要在运行时追踪对象引用、标记-清除内存或调整堆结构，可能导致不可预测的停顿（Stop-the-World），影响实时性、高吞吐场景的性能

## 如何回收

在变量离开作用域之后自动调用drop 方法进行回收

## 把一个旧变量赋予一个新变量
rust stack 和heap 很不一样, 不像其他语言区分没这么大.

* 当发生在stack 上

所有的copy 都是deep copy, 例如int , bool 等变量

* 当发生在heap 上

除非你显示调用clone , 都是move. 

