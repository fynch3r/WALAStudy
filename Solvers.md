WALA 包括多种类型的求解器，适用于各种风格的数据流分析。 

# Simple Fixed-point 不动点
最简单的求解器是 Fixed-point 求解器 DefaultFixedPointSolver。 该求解器仅管理 Statement 工作列表，并 evaluate 这些语句，直到 solution 没有进一步更改。

使用 WALA 求解器的主要实现好处是语句之间的依赖关系的可以通过空间有效表示，以及基于拓扑顺序的各种近似评估语句的能力，后一个特点是对于某些问题的快速收敛至关重要。

WALA 的默认 IR 构造和 flow-insensitive 指针分析以及其他几种分析在最低级别，使用通用定点求解器。

# Killdall-Style Frameworks
com.ibm.wala.dataflow 包在 Fixed-point 求解器之上提供了一个层，以促进 Killdall 风格的数据流框架的实现。 这里的抽象有助于建立一个在图的节点 and/or 边上的数据流系统。

要解决反向数据流问题，只需使用 GraphInverter.invert() 反转图中的边，并适当地设置 flow functions。

# RHS Solver
TabulationSolver 为 Sharir-Pneuli 风格的方法 context-sensitive 数据流分析（包括 IFDS）提供了 RHS 算法的实现。 TabulationSolver 的实现已经针对空间效率进行了调整，并包括各种扩展以帮助实现异常控制流等功能。

要解决反向数据流问题，请从 BackwardsSupergraph 开始。
