

探秘Metaspace
===========
> sczyh30,Eric Zhao，2015-06-26

Java 8 彻底将`永久代(PermGen)`移除出了 HotSpot JVM，将其原有的数据迁移至 Java Heap 或 Metaspace。
这一篇文章我们来总结一下**Metaspace（元空间）的特性**。


## 引言：永久代为什么被移出HotSpot JVM了？
在 HotSpot JVM 中，**永久代**中用于**存放类和方法的元数据以及常量池**，比如`Class`和`Method`。
每当一个类初次被加载的时候，它的元数据都会放到永久代中。

`永久代是有大小限制的`，因此如果加载的类太多，很有可能导致`永久代内存溢出`，
即万恶的 java.lang.`OutOfMemoryError: PermGen error`，为此我们不得不对虚拟机做调优。

那么，`Java 8 中 PermGen 为什么被移出 HotSpot JVM 了？`
我总结了两个主要原因（详见：JEP 122: Remove the Permanent Generation）：
1. 由于 `PermGen 内存经常会溢出`，引发恼人的 java.lang.`OutOfMemoryError: PermGen error`，
   因此 JVM 的开发者希望这一块内存可以更灵活地被管理，不要再经常出现这样的 OOM。
2. 移除 PermGen 可以促进 HotSpot JVM 与 JRockit VM 的融合，因为 JRockit 没有永久代。

根据上面的各种原因，PermGen 最终被移除，**方法区移至 Metaspace，字符串常量移至 Java Heap。**


## 探秘元空间
由于 Metaspace 的资料比较少，这里主要是依据**Oracle官方的Java虚拟机规范及Oracle Blog里的几篇文章**来总结的。

首先，**Metaspace（元空间）是哪一块区域？** 官方的解释是：
> In JDK 8, **classes metadata** is now stored in the **native heap** and this space is called **Metaspace**.

JDK 8 开始把**类的元数据**放到**本地堆内存(native heap)** 中，这一块区域就叫 **Metaspace**，中文名叫**元空间**。

### 优点
**使用本地内存有什么好处呢？**
最直接的表现就是`OOM问题将不复存在`，因为默认的`类的元数据分配只受本地内存大小的限制`，
也就是说本地内存剩余多少，理论上Metaspace就可以有多大（貌似容量还与操作系统的虚拟内存有关？这里不太清楚），这解决了`空间不足的问题`。
不过，让 Metaspace 变得无限大显然是不现实的，因此我们也要`限制 Metaspace 的大小`：
使用 **-XX:MaxMetaspaceSize** 参数来**指定 Metaspace 区域的大小**。
`JVM 默认在运行时根据需要动态地设置 MaxMetaspaceSize 的大小。`

除此之外，它还有以下优点：
* Take advantage of Java Language Specification property: Classes and associated metadata lifetimes match class loader’s
* Linear allocation only/仅限线性分配
* No individual reclamation (except for RedefineClasses and class loading failure)
* No GC scan or compaction/没有GC扫描或压缩
* No relocation for metaspace objects/没有对元空间对象的重定位

### GC
如果Metaspace的空间占用达到了设定的最大值，那么就会触发GC来收集死亡对象和类的加载器。
`根据JDK 8的特性，G1和CMS都会很好地收集Metaspace区（一般都伴随着Full GC）。`

为了减少垃圾回收的频率及时间，控制吞吐量，对Metaspace进行适当的监控和调优是非常有必要的。
如果`在Metaspace区发生了频繁的Full GC`，那么可能表示存在`内存泄露或Metaspace区的空间太小`了。

### 新增的JVM参数
* **-XX:MetaspaceSize** 是分配给**类元数据空间（字节）的初始大小**
  (Oracle逻辑存储上的初始高水位，the initial high-water-mark )，此值为估计值。
  `MetaspaceSize的值设置的过大会延长垃圾回收时间`。垃圾回收过后，引起下一次垃圾回收的类元数据空间的大小可能会变大。
* **-XX:MaxMetaspaceSize** 是分配给**类元数据空间的最大值**，`超过此值就会触发Full GC`，
  此值默认没有限制，但应取决于系统内存的大小。JVM会动态地改变此值。
* **-XX:MinMetaspaceFreeRatio** 表示一次GC以后，为了避免增加元数据空间的大小，空闲的类元数据的容量的最小比例，不够就会导致垃圾回收。
* **-XX:MaxMetaspaceFreeRatio** 表示一次GC以后，为了避免增加元数据空间的大小，空闲的类元数据的容量的最大比例，不够就会导致垃圾回收。

### 监控与调优（待补充）
使用 `ManagementFactory.getMemoryPoolMXBeans()` 返回的 [MemoryPoolMXBean](https://docs.oracle.com/javase/8/docs/api/java/lang/management/MemoryPoolMXBean.html)
可以监测 Metaspace 的动态，后续将更新这里。

`jstat -gc <JAVA_PID>`
* MC: Metaspace capacity (kB)
* MU: Metaspace utilization (kB)

`jmap –heap <JAVA_PID>`
* MetaspaceSize


## 参考资料
* The Java Virtual Machine Specification, Java SE 8 Edition, Oracle
  ([HTML](https://docs.oracle.com/javase/specs/jvms/se8/html/index.html) | [PDF](https://docs.oracle.com/javase/specs/jvms/se8/jvms8.pdf))
* [Metaspace in Java 8](https://java-latte.blogspot.com/2014/03/metaspace-in-java-8.html)
* [What is the use of Metaspace in Java 8? - StackOverflow](https://stackoverflow.com/questions/24074164/what-is-the-use-of-metaspace-in-java-8)
* [About G1 Garbage Collector, Permanent Generation and Metaspace](https://blogs.oracle.com/poonam/entry/about_g1_garbage_collector_permanent)
* [JEP 122: Remove the Permanent Generation](http://openjdk.java.net/jeps/122)
* [PermGen elimination in JDK 8 - StackOverflow](https://stackoverflow.com/questions/18339707/permgen-elimination-in-jdk-8/22509753#22509753)
* [Java 8 find out size of Metaspace at runtime - StackOverflow](https://stackoverflow.com/questions/31010463/java-8-find-out-size-of-metaspace-at-runtime)


[原文](https://www.sczyh30.com/posts/Java/jvm-metaspace/)

