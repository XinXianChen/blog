### 一 堆溢出

这种场景最为常见，报错信息：
```
java.lang.OutOfMemoryError: Java heap space
```

#### 原因
1、代码中可能存在大对象分配 

2、可能存在内存泄露，导致在多次GC之后，还是无法找到一块足够大的内存容纳当前对象。

#### 解决方法

1、检查是否存在大对象的分配，最有可能的是大数组分配 

2、通过jmap命令，把堆内存dump下来，使用内存离线工具分析一下（如mat），检查是否存在内存泄露的问题 

3、如果没有找到明显的内存泄露，使用 -Xmx 加大堆内存 4、还有一点容易被忽略，检查是否有大量的自定义的 Finalizable 对象，也有可能是框架内部提供的，考虑其存在的必要性

### 二 永久代/元空间溢出
```
java.lang.OutOfMemoryError: PermGen space
java.lang.OutOfMemoryError: Metaspace
```
#### 原因

永久代是 HotSot 虚拟机对方法区的具体实现，存放了被虚拟机加载的类信息、常量、静态变量、JIT编译后的代码等。

JDK8后，元空间替换了永久代，元空间使用的是本地内存，还有其它细节变化：

 - 字符串常量由永久代转移到堆中(jdk7就已完成)
 - 和永久代相关的JVM参数已移除
 - 可能原因有如下几种：
   1.在Java7之前，频繁的错误使用String.intern()方法
   
   2.运行期间生成了大量的代理类，导致方法区被撑爆，无法卸载
   
   3.应用长时间运行，没有重启
   
#### 解决方法

1.检查是否永久代空间或者元空间设置的过小

2.检查代码中是否存在大量的反射操作

3.dump之后通过mat检查是否存在大量由于反射生成的代理类

4.放大招，重启JVM

### 三 GC overhead limit exceeded
```
java.lang.OutOfMemoryError：GC overhead limit exceeded

```

#### 原因
这个是JDK6新加的错误类型，一般都是堆太小导致的。Sun 官方对此的定义：超过98%的时间用来做GC并且回收了不到2%的堆内存时会抛出此异常。

#### 解决方法

1.检查项目中是否有大量的死循环或有使用大内存的代码，优化代码。

2.添加参数 -XX:-UseGCOverheadLimit 禁用这个检查，其实这个参数解决不了内存问题，只是把错误的信息延后，最终出现 java.lang.OutOfMemoryError: Java heap space。

3.dump内存，检查是否存在内存泄露，如果没有，加大内存。

### 四 方法栈溢出 （无法创建线程）
```
java.lang.OutOfMemoryError : unable to create new native Thread

```

#### 原因
出现这种异常，基本上都是创建的了大量的线程导致的，以前碰到过一次，通过jstack出来一共8000多个线程。

#### 解决方法
1.通过 -Xss 降低的每个线程栈大小的容量

2.线程总数也受到系统空闲内存和操作系统的限制，检查是否该系统下有此限制：

- /proc/sys/kernel/pid_max
- /proc/sys/kernel/thread-max
- maxuserprocess（ulimit -u）
- /proc/sys/vm/maxmapcount

### 五 非常规溢出
#### 分配超大数组
```
java.lang.OutOfMemoryError: Requested array size exceeds VM limit


```
这种情况一般是由于不合理的数组分配请求导致的，在为数组分配内存之前，JVM 会执行一项检查。要分配的数组在该平台是否可以寻址(addressable)，如果不能寻址(addressable)就会抛出这个错误。

解决方法就是检查你的代码中是否有创建超大数组的地方。


#### swap溢出
```
java.lang.OutOfMemoryError: Out of swap space

```
这种情况一般是操作系统导致的，可能的原因有：

1.swap 分区大小分配不足；

2.其他进程消耗了所有的内存。

**解决方案**

1.其它服务进程可以选择性的拆分出去

2.加大swap分区大小，或者加大机器内存大小

     
#### 本地方法溢出
     
```
java.lang.OutOfMemoryError: stack_trace_with_native_method

```
本地方法在运行时出现了内存分配失败，和之前的方法栈溢出不同，方法栈溢出发生在 JVM 代码层面，而本地方法溢出发生在JNI代码或本地方法处。


     



