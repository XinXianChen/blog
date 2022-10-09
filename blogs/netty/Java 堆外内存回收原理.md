### 1、堆外内存简介
DirectByteBuffer 这个类是 JDK 提供使用堆外内存的一种途径，当然常见的业务开发一般不会接触到，即使涉及到也可能是框架（如 Netty、RPC 等）使用的，对框架使用者来说也是透明的

### 2、堆外内存优势
堆外内存优势主要体现在 IO 操作上，对于网络 IO，使用 Socket 发送数据时，能够节省java堆内存到堆外内存的数据拷贝，所以性能更高。看过 Netty 源码的同学应该了解，Netty 使用堆外内存来实现零拷贝技术。对于磁盘 IO 时，也可以使用内存映射，来提升性能。另外，更重要的几乎不用考虑堆内存GC 问题。

### 3、堆外内存创建
直接从代码来看，首先向 Bits 类申请额度，Bits 类内部维护着当前已经使用的堆外内存值，会 check 当前申请的大小与实际已经使用的内存大小是否超过总的堆外内存大小（默认大小与堆内存差不多，其实是有细微区别的，拿 CMS GC 来举例，它的大小是【新生代的最大值 - 一个 survivor 的大小 + 老生代的最大值】，可以使用
【 -XX:MaxDirectMemorySize】 参数指定堆外内存最大大小。
如果 check 不通过，会主动执行 System.gc()，然后 sleep 100 毫秒，再进行 check，如果内存还是不足，就抛出 OOM Error。

一、在默认情况下，通过System.gc()或者Runtime.getRuntime().gc()的调用，会显示触发Full GC，同时对老年代和新生代进行回收，尝试释放被丢弃对象占用的内存。

二、然而System.gc()调用附带一个免责声明，无法保证对垃圾收集器的调用。

三、JVM实现者可以通过System.gc()调用来决定JVM的GC行为。而一般情况下，垃圾回收应该是自动进行的，无须手动触发，否则就太过于麻烦了，在一些特殊情况下，如我们正在编写一个性能基准，我们可以在运行之间调用System.gc()。

如果 check 通过，就会调用 unsafe.allocateMemory(size) 真正分配内存，返回内存地址，然后再将内存清 0。这个 unsafe 不是说不安全，而是 JDK 内部使用的类，不推荐外部使用，所以叫 unsafe，Netty 源码内部也有类似命名。

由于申请内存前可能会调用 System.gc()，所以谨慎设置 -XX:+DisableExplicitGC 这个选项，这个参数作用是禁止代码中显示触发的 Full GC。

### 4、堆外内存回收
DirectByteBuffer内部维护了一个 Cleaner 对象的链表，通过 create(Object, Runnable) 方法创建 cleaner 对象，调用自身的 add 方法，将其加入到链表中。更重要的是提供了 clean 方法，clean 方法首先将对象自身从链表中删除，保证只调用一次，然后执行 this.thunk 的 run 方法，thunk 就是由创建时传入的 Runnable 参数，也就是说 clean 只负责触发 Runnable 的 run 方法，至于 Runnable 做什么任务它不关心。
#### 自动回收
Java 是不用用户去管理内存的，所以 Java 对堆外内存 默认是自动回收的。它是 由 GC 模块负责的，在 GC 时会扫描 DirectByteBuffer 对象是否有有效引用指向该对象，如没有，在回收 DirectByteBuffer 对象的同时且会回收其占用的堆外内存。但是 JVM 如何释放其占用的堆外内存呢？如何跟 Cleaner 关联起来呢？

这得从 Cleaner 继承了 PhantomReference（虚引用） 说起。说到 Reference，还有 SoftReference、WeakReference、FinalReference 他们作用各不相同，这里就不展开说了。

简单介绍 PhantomReference（虚引用），首先虚引用是不会影响 JVM 去回收其指向的对象，当 GC 某个对象时，如果有此对象上还有虚引用对其引用，会将 PhantomReference 对象插入 ReferenceQueue 队列。

PhantomReference插入到哪个队列呢？看 PhantomReference 类代码，其继承自 Reference，Reference 对象有个 ReferenceQueue 成员，这个也就是 PhantomReference 对象插入的 ReferenceQueue 队列，此成员如果不由外部传入就是 ReferenceQueue.NULL。如果需要通过 queue 拿到 PhantomReference 对象，这个 ReferenceQueue 对象还是必须由外部传入。

Reference 类内部 static 静态块会启动 ReferenceHandler 线程，线程优先级很高，这个线程是用来处理 JVM 在 GC 过程中交接过来的 reference。
```
static boolean tryHandlePending(boolean waitForNotify) {
        Reference<Object> r;
        Cleaner c;
        try {
            synchronized (lock) {
                if (pending != null) {
                    r = pending;
                    // 'instanceof' might throw OutOfMemoryError sometimes
                    // so do this before un-linking 'r' from the 'pending' chain...
                    c = r instanceof Cleaner ? (Cleaner) r : null;
                    // unlink 'r' from 'pending' chain
                    pending = r.discovered;
                    r.discovered = null;
                } else {
                    // The waiting on the lock may cause an OutOfMemoryError
                    // because it may try to allocate exception objects.
                    if (waitForNotify) {
                        lock.wait();
                    }
                    // retry if waited
                    return waitForNotify;
                }
            }
        } catch (OutOfMemoryError x) {
            // Give other threads CPU time so they hopefully drop some live references
            // and GC reclaims some space.
            // Also prevent CPU intensive spinning in case 'r instanceof Cleaner' above
            // persistently throws OOME for some time...
            Thread.yield();
            // retry
            return true;
        } catch (InterruptedException x) {
            // retry
            return true;
        }

        // Fast path for cleaners
        if (c != null) {
            c.clean();
            return true;
        }

        ReferenceQueue<? super Object> q = r.queue;
        if (q != ReferenceQueue.NULL) q.enqueue(r);
        return true;
    }
```
我们来看看 ReferenceHandler 是如何处理的？继承Thread类，直接看 run 方法，首先是个死循环，一直在那不停干活，synchronized 块内的这段主要是交接 JVM 扔过来的 reference（就是 pending），再往下看很明显，调用了 cleaner 的 clean 方法。调完之后直接 continue 结束此次循环，这个 reference 并没有进入 queue，也就是说 Cleaner 虚引用是不放入 ReferenceQueue。

#### 7、如何手动回收？
手动回收，就是由开发手动调用 DirectByteBuffer 的 cleaner 的 clean 方法来释放空间。由于 cleaner 是 private 权限，所以自然想到使用反射来实现。
还有另一种方法，DirectByteBuffer 实现了 DirectBuffer 接口，这个接口有 cleaner 方法可以获取 cleaner 对象。
Netty 中的堆外内存就是使用反射来实现手动回收方式进行回收的。



