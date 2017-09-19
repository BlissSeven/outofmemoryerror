# java.lang.OutOfMemoryError: **Metaspace**

# OutOfMemoryError系列（4）: Metaspace

Java applications are allowed to use only a limited amount of memory. The exact amount of memory your particular application can use is specified during application startup. To make things more complex, Java memory is separated into different regions, as seen in the following figure:

JVM有最大内存限制, 通过修改启动参数可以改变这些值。Java将堆内存划分为多个部分, 如下图所示:


![metaspace error](04_01_OOM-example-metaspace.png)



The size of all those regions, including the metaspace area, can be specified during the JVM launch. If you do not determine the sizes yourself, platform-specific defaults will be used.

这些区域的最大值, 由JVM启动参数 `-Xmx` 和 `-XX:MaxMetaspaceSize` 指定. 如果没有明确指定, 则根据平台类型(OS版本+ JVM版本)和物理内存的大小来确定。

The _java.lang.OutOfMemoryError: Metaspace_ message indicates that the Metaspace area in memory is exhausted.

 _java.lang.OutOfMemoryError: Metaspace_ 错误信息所表达的意思是: **元信息区(Metaspace) 内存已被用满** 

## What is causing it?

## 原因分析

If you are not a newcomer to the Java landscape, you might be familiar with another concept in Java memory management called PermGen. Starting from Java 8, the memory model in Java was significantly changed. A new memory area called Metaspace was introduced and Permgen was removed. This change was made due to variety of reasons, including but not limited to:

如果对Java比较熟悉, 应该知道有一种叫做 PermGen 的内存区. 从 Java 8开始, Java的内存模型发生了重大改变。 引入了一个新的内存区 Metaspace, 用来替代 Permgen. 这种变化是基于多方面的考虑, 例如:

*   The required size of permgen was hard to predict. It resulted in either under-provisioning triggering [java.lang.OutOfMemoryError: Permgen size](http://www.plumbr.eu/outofmemoryerror/permgen-space) errors or over-provisioning resulting in wasted resources.

*   permgen具体需要设置多大是很难预测的。设置少了会造成 [java.lang.OutOfMemoryError: Permgen size](http://www.plumbr.eu/outofmemoryerror/permgen-space) 错误, 设置多了又会浪费。

*   [GC performance](https://plumbr.eu/handbook/gc-tuning/gc-tuning-in-practice) improvements, enabling concurrent class data de-allocation without [GC pauses](https://plumbr.eu/handbook/garbage-collection-algorithms-implementations) and specific iterators on metadata

*   [GC 性能](http://blog.csdn.net/renfufei/article/details/61924893) 的提升, 使得并发的垃圾收集过程中没有 [GC暂停](http://blog.csdn.net/renfufei/article/details/54885190), 对 metadata 的特殊迭代规则。

*   Support for further optimizations such as [G1](https://plumbr.eu/handbook/garbage-collection-algorithms-implementations/g1) concurrent class unloading.

*   支持对 [G1垃圾收集器](http://blog.csdn.net/renfufei/article/details/54885190#t9) 的深入优化, 以及并发执行 class 卸载。

So if you were familiar with PermGen then all you need to know as background is that – whatever was in PermGen before Java 8 (name and fields of the class, methods of a class with the bytecode of the methods, constant pool, JIT optimizations etc) – is now located in Metaspace.

对应的, PermGen 里的所有内容, 在Java8中都迁移到了 Metaspace 空间。例如, class 名称, 字段, 方法, 字节码, 常量池, JIT优化后的代码等等。

As you can see, Metaspace size requirements depend both upon the number of classes loaded as well as the size of such class declarations. So it is easy to see the **main cause for the _java.lang.OutOfMemoryError: Metaspace_ is: either too many classes or too big classes being loaded to the Metaspace.**

如您所见, Metaspace 的使用量和JVM加载到内存中的 class 数量/大小有关。可以说 _java.lang.OutOfMemoryError: Metaspace_ 错误的主要原因, 是加载到内存中的 class 数量太多或体积太大。

## Give me an example

## 示例

As we explained in the previous chapter, Metaspace usage is strongly correlated with the number of classes loaded into the JVM. The following code serves as the most straightforward example:

和上一章讲的类似, Metaspace 空间的使用量, 与JVM加载的 class 数量有很大关系。下面是一个简单的例子:

```
public class Metaspace {
  static javassist.ClassPool cp = javassist.ClassPool.getDefault();

  public static void main(String[] args) throws Exception{
    for (int i = 0; ; i++) { 
      Class c = cp.makeClass("eu.plumbr.demo.Generated" + i).toClass();
    }
  }
}

```



In this example the source code iterates over a loop and generates classes at the runtime. All those generated class definitions end up consuming Metaspace. Class generation complexity is taken care of by the [javassist](http://www.csg.ci.i.u-tokyo.ac.jp/~chiba/javassist/) library.

可以看到, 使用 [javassist](http://www.csg.ci.i.u-tokyo.ac.jp/~chiba/javassist/) 工具类生成 class 是非常简单的。这段代码在 for 循环中, 动态生成了很多class。这些生成的 class 最终会被加载到 Metaspace 中。

The code will keep generating new classes and loading their definitions to Metaspace until the space is fully utilized and the _java.lang.OutOfMemoryError: Metaspace_ is thrown. When launched with _-XX:MaxMetaspaceSize=64m_ then on Mac OS X my Java 1.8.0_05 dies at around 70,000 classes loaded.

执行这段代码, 会生成很多新的 class 并将其加载到内存中, 随着生成的class越来越多,将会占满 Metaspace 空间, 然后抛出 _java.lang.OutOfMemoryError: Metaspace_. 如果设置了启动参数 _-XX:MaxMetaspaceSize=64m_, 在Mac OS X上, Java 1.8.0_05 大约加载了 70000 个class后挂掉。

## What is the solution?

## 解决方案

The first solution when facing the OutOfMemoryError due to Metaspace should be obvious. If the application exhausts the Metaspace area in the memory you should increase the size of Metaspace. Alter your application launch configuration and increase the following:

面对 Metaspace 方面的 OutOfMemoryError 时, 第一大解决方案是增加 Metaspace 的大小. 使用类似下面这样的启动参数即可:

```
-XX:MaxMetaspaceSize=512m
```


The above configuration example tells the JVM that Metaspace is allowed to grow up to 512 MB before it can start complaining in the form of _OutOfMemoryError_.

以上配置将JVM的最大 Metaspace 设置为 512MB, 如果还不够, 就会抛出 _OutOfMemoryError_。

Another solution is even simpler at first sight. You can remove the limit on Metaspace size altogether by deleting this parameter. But pay attention to the fact that by doing so you can introduce heavy swapping and/or reach native allocation failures instead.

有一种看起来很简单的解决方案, 就是直接去掉 Metaspace 的大小限制。 但需要注意, 不限制Metaspace内存, 如果物理内存不足, 就有可能产生内存交换(swapping), 严重拖累系统性能。 当然,还可能造成native内存分配失败。

> 我们知道,在现代web集群中,应用系统宁可死,也不可慢。

Before calling it a night though, be warned – more often than not it can happen that by using the above recommended “quick fixes” you end up masking the symptoms by hiding the _java.lang.OutOfMemoryError: Metaspace_ and not tackling the underlying problem. If your application leaks memory or just loads something unreasonable into Metaspace the above solution will not actually improve anything, it will just postpone the problem.

如果不想收到报警, 可以像鸵鸟一样, 把 _java.lang.OutOfMemoryError: Metaspace_ 错误信息隐藏起来。 但这不能真正解决问题, 只会推迟问题爆发的时间。 如果确实存在内存泄露, 请参考前面的文章, 认真解决问题。


原文链接: <https://plumbr.eu/outofmemoryerror/permgen-space>



