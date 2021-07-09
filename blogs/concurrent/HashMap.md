## HashMap
### 数据结构
1.7 = 数组（基础） + 链表 
（>=）1.8 = 数组 + 链表 + 红黑树 （引入红黑树！容量>=64才会链表转红黑树，否则优先扩容只有等链表过长，阈值设置TREEIFY_THRESHOLD = 8，不是代表链表长度，链表长度>8,链表9的时候转红黑树）

### 数组的大小

new HashMap（）；如果不写构造参数，默认大小16

如果说：写了初始容量：11 ？hashmap的容量就是11？

hashmap的get，put操作时间复杂度O(1)



### 数组所有的元素位是否能够100%被利用起来？

不一定，hash碰

引入链表结构解决hash冲突，采用头部插入链表法，链表时间复杂度O(n)


### length的限制
/** * The default initial capacity - MUST be a power of two. */必须是2的指数幂？
roundUpToPowerOf2(size)，强型将非2的指数次幂的数值转化成2的指数次幂
怎么转化？
1、必须最接近size，11
2、必须大于=size，
3、是2的指数次幂
16
size = 17，capacity = 32

### 为什么一定要转成2的指数次幂？
计算索引时用到了&运算，只有是2的指数次的时候，length - 1 时二进制上都是1
```
int i = indexFor(hash, table.length);
static int indexFor(int h, int length) {
//  key.hashCode % table.lenth
	return h & (table.lenth-1);
}

h = 
0001 0101 0111 0010 1111
0001 0101 0000 0010 0000
16
    0
    
0000 0000 0000 0000 1111 16-1=15
0000 0000 0000 0000 1010
0-15
bit位运算：1815ms
mod取模运算：22282
效率差10倍

```


### HashMap扩容
当前hashmap存了多少element，size>=threshold
threshold扩容阈值 = capacity * 扩容阈值比率 0.75 = 16*0.75=12
扩容怎么扩？
扩容为原来的2倍。

hash扩容，有个加载因子？loadfactor = 0.75为什么是0.75
0.5
1
牛顿二项式：基于空间与时间的折中考虑0.5

### Jdk7-扩容死锁分析 
死锁问题核心在于下面代码，多线程扩容导致形成的链表环!
```
void transfer(Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length;
        for (Entry<K,V> e : table) {
            while(null != e) {
                Entry<K,V> next = e.next;
                if (rehash) { 
                    e.hash = null == e.key ? 0 : hash(e.key);//再一次进行hash计算？
                }
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            }
        }
    }
```

### Jdk8-扩容
Java8 HashMap扩容跳过了Jdk7扩容的坑，对源码进行了优化，采用高低位拆分转移方 式，避免了链表环的产生。
```
Node<K,V> loHead = null, loTail = null;
Node<K,V> hiHead = null, hiTail = null;
Node<K,V> next;
    do {
        next = e.next;
        if ((e.hash & oldCap) == 0) {
            //yangguo.hashcode & 16 = 0，用低位指针
            if (loTail == null)
                loHead = e;
            else
                loTail.next = e;
            loTail = e;
        }
        else {
             //yangguo.hashcode & 16 》 0 高位指针
            if (hiTail == null)
                hiHead = e;
            else
                hiTail.next = e;
            hiTail = e;
        }
    } while ((e = next) != null);
if (loTail != null) {
    loTail.next = null; 
    newTab[j] = loHead;，移到新的数组上的同样的index位置
}
if (hiTail != null) {
    hiTail.next = null;
    newTab[j + oldCap] = hiHead; //index 3+16 = 19
}

```
完全绕开rehash，要满足高低位移动，必须数组容量是2的幂次方

## ConcurrentHashMap
### 数据结构
ConcurrentHashMap的数据结构与HashMap基本类似，区别在于:

    -  内部在数据写入时加了同步机制(分段锁)保证线程安全，读操作是无锁操作。
    -  扩容时老数据的转移是并发执行的，这样扩容的效率更高。
    
### 并发安全控制
Java7 ConcurrentHashMap基于ReentrantLock实现分段锁

Java8 ConcurrentHashMap基于分段锁+CAS保证线程安全，分段锁基于synchronized 关键字实现;

#### 重要成员变量

    - LOAD_FACTOR: 负载因子, 默认75%, 当table使用率达到75%时, 为减少table 的hash碰撞, tabel长度将扩容一倍。负载因子计算: 元素总个数%table.lengh
    - TREEIFY_THRESHOLD: 默认8, 当链表长度达到8时, 将结构转变为红黑树。
    - UNTREEIFY_THRESHOLD: 默认6, 红黑树转变为链表的阈值。
    - MIN_TRANSFER_STRIDE: 默认16, table扩容时, 每个线程最少迁移table的槽位个数。
    - MOVED: 值为-1, 当Node.hash为MOVED时, 代表着table正在扩容
    - TREEBIN, 置为-2, 代表此元素后接红黑树。
    - nextTable: table迁移过程临时变量, 在迁移过程中将元素全部迁移到nextTable上。
    - sizeCtl: 用来标志table初始化和扩容的,不同的取值代表着不同的含义:0: table还没有被初始化；-1: table正在初始化；小于-1: 实际值为resizeStamp(n)<<RESIZE_STAMP_SHIFT+2, 表明table正在扩容；大于0: 初始化完成后, 代表table最大存放元素的个数, 默认为0.75*n
    - transferIndex: table容量从n扩到2n时, 是从索引n->1的元素开始迁移, transferIndex代表当前已经迁移的元素下标
    - ForwardingNode: 一个特殊的Node节点, 其hashcode=MOVED, 代表着此时 table正在做扩容操作。扩容期间, 若table某个元素为null, 那么该元素设置为 ForwardingNode, 当下个线程向这个元素插入数据时, 检查hashcode=MOVED, 就 会帮着扩容。

ConcurrentHashMap由三部分构成, table+链表+红黑树, 其中table是一个数组, 既然是 数组, 必须要在使用时确定数组的大小, 当table存放的元素过多时, 就需要扩容, 以减少碰撞 发生次数, 本文就讲解扩容的过程。扩容检查主要发生在插入元素(putVal())的过程:
    
    - 一个线程插完元素后, 检查table使用率, 若超过阈值, 调用transfer进行扩容
    - 一个线程插入数据时, 发现table对应元素的hash=MOVED, 那么调用helpTransfer()协助扩容。
    - table扩容过程就是将table元素迁移到新的table上, 在元素迁移时, 可以并发完成, 加快 了迁移速度, 同时不至于阻塞线程。所有元素迁移完成后, 旧的table直接丢失, 直接使用新的 table。
        
    


    