一些提高 WALA 分析效率和结果准确性的 tips。

# Performance Tips

## Heap Size
使用 -Xmx<n>M 将堆大小设置为 n 兆字节。 使用 -verbose:gc 查看 VM 是否处于 Discontent GC 区域。

想知道内存的去向吗？ 查看 HeapTracer 工具类。 它很慢，但它很有用。

## Libraries
在分析 JDK 的早期版本（例如 1.4）时，许多分析（例如调用图构造）可能会运行得更快，因为 5.0+ 库引入了我们尚未追踪的各种污染。 您可能希望针对 1.4 JDK 对您的分析进行原型设计，然后升级到更高版本，可以后续使用 Exclusions 文件来恢复失去的性能。

## Exclusions File
WALA 支持基于 XML 格式的 exclusions file ，我们认为使用 exclusions file 可以大大加快分析速度，尽管当然会引入潜在的不健全性。

例如，您可以使用以下代码（来自 CallGraphTestUtil）构建基于排除文件排除类的分析范围。

```java
public static AnalysisScope makeJ2SEAnalysisScope(String scopeFile, String exclusionsFile) {
  AnalysisScope scope = AnalysisScopeReader.read(scopeFile, exclusionsFile, MY_CLASSLOADER);
  return scope;
}
```

下面是 GUIExclusions.txt 的内容，这是一个exclusions file，它告诉 WALA 忽略与 AWT 相关的类。 使用这些排除项时，WALA 会假装 java.awt 包中的类不存在。

[CallGraphTest.testPrimordial() 测试](https://github.com/wala/WALA/blob/95fde985336f6e6d6c72a18418b7f82535544ea4/com.ibm.wala.core.tests/src/com/ibm/wala/core/tests/callGraph/CallGraphTest.java#L246)

下面是 GUIExclusions.txt 的内容，这是一个exclusions file，它告诉 WALA 忽略与 AWT 相关的类。 使用这些排除项时，WALA 会假装 java.awt 包中的类不存在。
```txt
java\/awt\/.*
javax\/swing\/.*
sun\/awt\/.*
sun\/swing\/.*
```
