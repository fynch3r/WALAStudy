# 仓库依赖配置
这里以 Maven 仓库配置为例，需要四个依赖：
- WALA Core
- WALA Util
- WALA Shrike
- WALA Cast

```xml
<dependencies>
        <!-- https://mvnrepository.com/artifact/com.ibm.wala/com.ibm.wala.core -->
        <dependency>
            <groupId>com.ibm.wala</groupId>
            <artifactId>com.ibm.wala.core</artifactId>
            <version>1.5.8</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/com.ibm.wala/com.ibm.wala.util -->
        <dependency>
            <groupId>com.ibm.wala</groupId>
            <artifactId>com.ibm.wala.util</artifactId>
            <version>1.5.8</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/com.ibm.wala/com.ibm.wala.shrike -->
        <dependency>
            <groupId>com.ibm.wala</groupId>
            <artifactId>com.ibm.wala.shrike</artifactId>
            <version>1.5.8</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/com.ibm.wala/com.ibm.wala.cast -->
        <dependency>
            <groupId>com.ibm.wala</groupId>
            <artifactId>com.ibm.wala.cast</artifactId>
            <version>1.5.8</version>
        </dependency>
</dependencies>
```
## 项目配置
WALA 推荐用户实现三个配置文件，建议均放在src/main/resources目录下：
- scope.txt
- exclusions.txt
- wala.properties

## scope.txt
分析域的配置文件，WALA提供了读取配置文件和直接添加类文件进入分析域两种构建分析域的手段 scope.txt属于读取配置文件的方式，参考 Analysis-Scope

## exclusions.txt
不需要分析的类，支持正则表达式，参考 exclusions.txt
主要为了给 CallGraph 的生成增强可拓展性（虽然这种行为 unsoundness

## wala.properties
WALA 属性配置文件，官方推荐初学者设置好java_runtime_dir和output_dir两个属性：
- java_runtime_dir: 具体到jdk的具体目录，Home 即可，不用深入到 bin 目录
- output_dir: 分析结果输出地址，必须事先创建好

