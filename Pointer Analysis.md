#Overview
WALA 提供了一个流不敏感的 Anderson 风格的指针分析框架，可以通过多种不同的方式实现上下文敏感的指针分析。

WALA 所有的指针分析实现都以构建 on-the-fly 函数调用图。

context-sensitivity 主要是体现在两个维度：
- HeapModel：指针分析模型是如何抽象指针和堆位置的？
- ContextSelector：Call Graph 的构建过程中如何跟据上下文克隆方法？

可以通过自定义堆模型和上下文选择器来定义自己的指针分析策略。 
此外，WALA 提供了许多内置策略。

# Heap Model 堆模型
一个 HeapModel 对象告诉 WALA 指针分析框架如何抽象指针和堆位置。关键类：
- PointerKey ：指针的抽象
- InstanceKey ：堆位置的抽象

举个例子，一个PointerKey 可以表示一个 local variable 、一个 static field、一个 instance field 等等。

一个PointerKey 是具体程序的若干指针的等价类的名称，这些指针在抽象中被整合到一个抽象对象中，也就是 PointerKey。

InstanceKey 可以表示一个具体类型的所有实例化对象，或者由一个特定的 allocation-site 创建的所有对象，或者在一个特定的 context 中特定 allocation-site 创建的所有对象。

HeapModel 为指针分析提供回调（call-backs），用来在分析期间创建 PointerKeys 和 InstanceKeys，可以通过提供自己的 HeapModel 来自定义策略。

# Heap Graph 堆图
一个 HeapGraph 对象提供了一种方便的方式用来指导指针分析的结果。HeapGraph 由 PointerKey 和 InstanceKey 组成。

如果指针分析表示一个 P 可能指向 I ，那么 在 HeapGraph 存在一条边，从 PointerKeyP 指向 InstanceKey I。

从  InstanceKey I  指向  PointerKeyP  ，当且仅当：
- P 代表一个具体对象的具体属性，这个具体对象由 P 定义；
- P 代表数组内容，这个数组实例由 I 代表

举个例子，给你一个 HeapGraph 对象 h，假设你想知道所有和PointerKey  p 构成别名关系的PointerKey 指针对象们。

你需要首先用h.getSuccNodes(p) 找出所有 p 可能指向的全部 InstanceKey I 。对于每个 InstanceKey I ，使用h.getPredNodes(i) 找到所有i的前驱指针，这些前驱指针可能就是 p 的别名指针。

# Context Selector 上下文选择器
在 Call Graph 中的每个 node 都表示特定 context 的一个方法。ContextSelector 控制调用图构建 context 的策略。

最简单的 context 就是Everywhere.EVERYWHERE ，它表示一个方法的单个全局上下文。

其他 context 策略可以表示 call-string contexts，命名返回对象的上下文以实现对象敏感性或其他变体。

你可以通过提供自定义的 ContextSelector 对象来自定义上下文敏感策略。


回想一下，每个调用图节点都代表特定上下文中的一个方法。 ContextSelector 对象控制调用图构建上下文的策略。

最简单的 Context 是 Everywhere.EVERYWHERE，它表示方法的单个全局上下文。

其他上下文策略可以表示调用字符串上下文、命名接收者对象的上下文以实现对象敏感性或其他变体。

您可以通过提供自定义 ContextSelector 对象来自定义上下文敏感策略。

# Entry Points 入口点
例如我们有一个方法作为公共入口点：
```java
public static void foo(Object x,Object y)
