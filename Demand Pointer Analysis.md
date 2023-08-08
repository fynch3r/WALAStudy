按需指针分析，DemandRefinementPointsTo

# Initialization 初始化
在 DemandRefinementPointsTo 对象的初始化期间必须设置几个参数，如下所述。 基本的初始化代码可以看 trunk.tests.demandpa 项目中的 AbstractPtrTest.makeDemandPointerAnalysis() 方法。

## CallGraph
在 DemandRefinementPointsTo 对象的初始化期间必须设置几个参数，如下所述。 基本的初始化代码可以看 trunk.tests.demandpa 项目中的 AbstractPtrTest.makeDemandPointerAnalysis() 方法。

## HeapModel
该分析还需要一个 HeapModel 来确定如何抽象指针和堆位置。 该分析主要使用通过调用 Util.makeVanillaZeroOneCFABuilder 来分析。

## State Machine
可以扩展分析以获得更高的精度或使用 StateMachine 跟踪其他属性。 例如，上下文敏感是通过使用 ContextSensitiveStateMachine 来实现的。 DemandRefinementPointsTo 类的构造函数将 StateMachineFactory 作为参数； 该工厂类用于为每个查询和每个细化通道创建一个新的 StateMachine 对象。 对于上下文不敏感的分析，传入一个 DummyStateMachine.Factory 对象。

## Refinement Policy
RefinementPolicy 规定分析如何处理字段访问和方法调用，还制定每个细化传递的预算。在默认情况下，分析使用SinglePassRefinementPolicy 来分析单次传递，没有 field-sensitive 和动态（on-the-fly）调用图细化。其他的分析细化策略包括：
- ManualRefinementPolicy ，它针对某些手动指定的字段（主要是 java.util 包下的集合类）进行细化；
- TunedRefinementPolicy ，它试图发现哪些 field 和方法调用对精确处理很重要。

与状态机类似，通过调用 DemandRefinementPointsTo.setRefinementPolicyFactory() 来设置RefinementPolicyFactory 来更改细化策略。

# Running the Analysis
运行分析最直接的方法是调用 DemandRefinementPointsTo.getPointsTo() 方法，它只接受一个 PointerKey 作为参数。 （PointerKey 应该从用于初始化分析的同一 HeapModel 中获得。）如果分析能够在 RefinementPolicy 指定的预算内计算结果，则返回该结果；否则，分析返回 null。

要对分析进行更多控制，请调用 DemandRefinementPointsTo.getPointsTo() 方法，将 PointerKey 和 Predicate 作为参数。 Predicate 指定了分析结果 point-to set 理论上应该满足的条件。 （例如，如果指向集用于检查 down-cast 转换的安全性，则谓词将检查集合中没有实例键会导致向下转换的失败 case。）该方法返回指向集（同样可能为 null）和一个 PointsToResult 指示是否：满足 (1) 谓词，(2) 不再可能进行细化，或 (3) 超出分析预算。
