#Overview

CallGraph 类通过方法的逻辑克隆，表示潜在的上下文敏感（context-sensitive）调用图。

每个调用图节点（CGNode）表示当前 Context 中的方法 IMethod。

那么什么是上下文？基本上，Context 只是一个名称，是 IMethod 的克隆。

对于上下文不敏感（context-insentive）的调用图，用 Everywhere.Everywhere 作为上下文不敏感算法中的默认上下文。

注意，对于给定的 IMethod，上下文敏感的调用图可能有许多表示该方法的节点（上下文）。我们可以使用：
`CallGraph.getNodes(MethodReference methodReference)` 方法来获取所有相关节点。

WALA 支持一系列即使调用图（on-the-fly）构建算法，与流不敏感的指针分析集成。有关详细信息，请参见：[指针分析](https://github.com/wala/WALA/wiki/Pointer-Analysis)。

WALA还通过快速类型分析（Rapid Type Analysis，RTA）实现了调用图构造（请参见 Util.makeRTABuilder()），但强烈建议不要使用它。

WALA 所包含的 0-CFA 分析几乎总是更快更精确（具体参考:[指针分析](https://github.com/wala/WALA/wiki/Pointer-Analysis)）。此外，RTA 对字节码进行操作，其中一些字节码可能在 SSA 构建期间被证明是 dead code，因此不会出现在 SSA IR 中参与表示。这可能导致不期望的行为，如 RTA 调用图返回的调用站点，而这些调用站点不会出现在SSA IR中。最后，请注意在 RTA 调用图中对例如 Object.clone()等原生方法的建模是 "sound enough" 的，以确保调用图包括所有可访问的代码，如果不这样的话可能会不健全。

# EntryPoint
确定入口点：
针对主程序的main方法作为入口点：`Util.makeMainEntryPoints()`
针对所有应用类方法均作为入口点：`AllApplicationEntrypoints()`

# Build Callgraph
- CHACallGraph: 使用 CHA 算法构建，精度低、速度快；
```java
CHACallGraph cg = new CHACallGraph(cha);
cg.init(new AllApplicationEntrypoints(scope,cha);
```
- 0-CFA: context-sensitive，精度高、数据慢
```java
ClassHierarchy cha = ClassHierarchyFactory.makeWithRoot(scope);
AllApplicationEntrypoints entrypoints = new AllApplicationEntrypoints(scope, cha);
AnalysisOptions option = new AnalysisOptions(scope, entrypoints);
SSAPropagationCallGraphBuilder builder = Util.makeZeroCFABuilder(
  Language.JAVA, option, new AnalysisCacheImpl(), cha, scope);
```
