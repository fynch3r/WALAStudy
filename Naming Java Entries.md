在 Java 字节码中，在运行时出现的单个给定实体可以有多个名称。

例如，你在 foo 包中有 A、B 两个类， A 是B 的子类，但两个类都没有重写 java.lang.Object.toString() 方法
那么下列合法名称都有可能在字节码中出现：
- < Application, Lfoo/A, toString() >
- < Application, Lfoo/B, toString() >
- < Application, Ljava/lang/Object, toString() >
- < Primordial, Ljava/lang/Object, toString() >

但是，在 JVM Runtime 环境下，只能有一个名称：<Primordial,Ljava/lang/Object,toString()>
在 WALA 中，存在多对组合可以表示如上关系：
MethodReference 可以表示字节码中可能出现的任何名称 而 IMethod 表示运行时的唯一方法实体
在 Java 中，类的唯一标识可以视为 classloader x packagename.typename 形式的组合。
TypeReference 可以表示一个类型在字节码时的任何名称，IClass 表示运行时确定的唯一类名称
因此，一个TypeReference可以由 ClassLoaderReference 和 TypeName 组成

这种关系同样适用于：
FieldReference 和 IField
ClassLoaderReference 和 IClassLoader
