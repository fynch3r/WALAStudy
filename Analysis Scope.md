# Analysis Scope
AnalysisScope 对象规范了待分析的应用以及 jar 包代码。

# 创建 AnalysisScope 对象
- 方式a:
```java
AnalysisScope scope = AnalysisScopeReader.instance.makeJavaBinaryAnalysisScope(String classPath, File exclusionsFile)
```
参数含义:
- classpath: 待分析代码的路径;
- exclusionsFile: 不需要分析的配置文件；

- 方式b:
```java
AnalysisScope scope = AnalysisScopeReader.instance.readJavaScope(String scopeFileName, File exclusionsFile, ClassLoader javaLoader)
```
参数含义：
- scopeFileName: scope.txt文件路径，如果已经写在src/main/resources路径下，那么写scope.txt即可。
- exclusionFile：不需要分析的配置文件
- javaLoader:：类加载器，xxx.class.getClassLoader()
  
# 修改 AnalysisScope 对象的分析氛围
```java
scope.addClassFileToScope(ClassLoaderReference loader, File file)
```
参数：
- loader: ClassLoaderReference.Application
- file: class 文件

当然还有很多方式：

<img width="702" alt="image" src="https://github.com/fynch3r/WALAStudy/assets/45999489/bf924d1c-e69f-414d-8d72-bac0cf9af6c6">

# scope.txt 内容
scope.txt 内容格式：
```txt
ClassLoader,Language,Type,Location
```
下面分别说一下这些字段的含义：

## ClassLoader 字段
WALA 将 Java 语言中的类加载器分为三种类型：
- Primordial: 用于分析 JDK 原生标准库。如果想使用与JVM WALA运行时相同的标准库进行分析，可以将下面两行写进 scope ：
```txt
Primordial,Java,stdlib,none
Primordial,Java,jarFile,primordial.jar.model
```
如果你想分析Java标准库的其他版本，请删除none行，改为添加Primordal,Java,stdlib,/path/to/lib.jar格式的行（例如，rt.jar文件）。保留Primordial,Java,jarFile,primordial.jar.model行，以便使用适当的库模型。
- Extension: 用于分析应用程序用到的其他库
- Application: 用于分析应用程序自身代码

## Language 字段
Java 即可

## Type 字段
- classFile: 表示单独的.class文件
- binaryDir: 表示包含若干.class文件的目录，具有包名-目录层级的对应结构
- jarFile: 表示一个.jar文件
- sourceFile/sourceDir: 用于分析.java源码，用于Java-source-front-end，或为范围内的类文件提供源文件位置。对于后一种用例，确保类文件/目录出现在 scope.txt 中相应的sourceFile或sourceDir之前。

## Location 字段
表示合适的文件路径：
```txt
Primordial,Java,stdlib,none
Primordial,Java,jarFile,primordial.jar.model
Extension,Java,jarFile,/workspace/myapp/lib/someLib.jar
Application,Java,binaryDir,/workspace/myApp/bin
```
