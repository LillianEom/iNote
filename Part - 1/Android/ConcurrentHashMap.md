# ConcurrentHashMap

（1）ConcurrentHashMap 的锁分段技术

（2）ConcurrentHashMap 的读是否要加锁，为什么

（3）ConcurrentHashMap 的迭代器是强一致性的迭代器还是弱一致性的迭代器

**ConcourrentHashMap 保证线程安全的方法是**：**分段锁技术**



### 使用ConCurrentHashMap的原因

#### **线程不安全的HashMap**

在多线程环境下，使用 HashMap 进行 put 操作会引起死循环，导致 CPU 利用率接近100%，所以在并发情况下不能使用 HashMap。HashMap 在并发执行 put 操作是会引起死循环，是因为多线程会导致 HashMap 的 Entry 链表形成环形数据结构，一旦形成环形数据结构，Entry 的 next 节点永远不为空，就会产生死循环。

#### **效率低下的HashTable**

HashTable 使用 synchronized 来保证线程的安全，但是在线程竞争激烈的情况下 HashTable 的效率非常低下。HashTable 只有一把锁，当一个线程访问 HashTable 的同步方法，会将整张 table  锁住，其他方法访问 HashTable 的同步方法时，会进入阻塞或者轮询状态。如果线程1使用 put 进行元素添加，线程2不但不能用 put 方法添加于元素同是也无法用 get 方法来获取元素，所以竞争越激烈效率越低。

#### **ConcurrentHashMap的锁分段技术**

HashTable 容器在竞争激烈的并发环境效率低下的原因是所有访问 HashTable 的线程都必须竞争同一把锁，假如容器有多把锁，每一把锁用于锁住容器中一部分数据，那么多线程访问容器里不同数据段的数据时，线程间就不会存在锁竞争，从而可以有效提高并发访问率，这就是ConcurrentHashMap 的锁分段技术。将数据分成一段一段的存储，然后给每一段数据配一把锁，当一个线程占用锁访问其中一段数据的时候，其他段的数据也能被其他线程访问。

### 1.7实现

底层采用数组 + 链表 + 红黑树的存储结构。

#### 数据结构

jdk1.7中采用`Segment` + `HashEntry`的方式进行实现，结构如下：

![img](http://upload-images.jianshu.io/upload_images/2184951-af57d9d50ae9f547.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`ConcurrentHashMap`初始化时，计算出`Segment`数组的大小`ssize`和每个`Segment`中`HashEntry`数组的大小`cap`，并初始化`Segment`数组的第一个元素；其中`ssize`大小为2的幂次方，默认为16，`cap`大小也是2的幂次方，最小值为2，最终结果根据根据初始化容量`initialCapacity`进行计算，计算过程如下：

```java
if (c * ssize < initialCapacity)
    ++c;
int cap = MIN_SEGMENT_TABLE_CAPACITY;
while (cap < c)
    cap <<= 1;
```

其中`Segment`在实现上继承了`ReentrantLock`，这样就自带了锁的功能。

#### 创建分段锁

ConcurrentHashMap  采用延迟初始化的模式，首先只 new 一个 Segment，剩下的 Segment 只在使用时初始化。每次 put 操作时，定位到 key 所在的 Segment 后，会先去判断 Segment 是否初始化了,没有的话由新建一个 Segment 并通过循环 CAS 设置 Segment 数组对应位置引用。

#### get()

get 方法不需要加锁操作，因为 HashEntry 中的 value 是 volatile，可以保证线程间的可见性。由于 segments 非 volatile，通过 UNSAFE 的 getObjectVolatile 方法提供 volatile 读语义来遍历获得对应链表上的节点。但没有锁可能会导致在遍历的过程中几点 被其它线程修改，返回的 val 可能是过时数据，这部分是 ConcurrentHashMap 非强一致性的体现，如果对强一致性有要求，那么就只能去使用 Hashtable 或 Collections.synchronizedMap 这种通过 synchronized 关键字进行加锁提供强一致性的 Map 容器。containsKey 操作与 get 是一样 的。

#### put实现

当执行`put`方法插入数据时，根据key的hash值，在`Segment`数组中找到相应的位置，如果相应位置的`Segment`还未初始化，则通过CAS进行赋值，接着执行`Segment`对象的`put`方法通过加锁机制插入数据，实现如下：

场景：线程A和线程B同时执行相同`Segment`对象的`put`方法

1、线程A执行`tryLock()`方法成功获取锁，则把`HashEntry`对象插入到相应的位置；
2、线程B获取锁失败，则执行`scanAndLockForPut()`方法，在`scanAndLockForPut`方法中，会通过重复执行`tryLock()`方法尝试获取锁，在多处理器环境下，重复次数为64，单处理器重复次数为1，当执行`tryLock()`方法的次数超过上限时，则执行`lock()`方法挂起线程B；
3、当线程A执行完插入操作时，会通过`unlock()`方法释放锁，接着唤醒线程B继续执行；

#### size实现

因为`ConcurrentHashMap`是可以并发插入数据的，所以在准确计算元素时存在一定的难度，一般的思路是统计每个`Segment`对象中的元素个数，然后进行累加，但是这种方式计算出来的结果并不一样的准确的，因为在计算后面几个`Segment`的元素个数时，已经计算过的`Segment`同时可能有数据的插入或则删除，在1.7的实现中，采用了如下方式：

```java
try {
    for (;;) {
        if (retries++ == RETRIES_BEFORE_LOCK) {
            for (int j = 0; j < segments.length; ++j)
                ensureSegment(j).lock(); // force creation
        }
        sum = 0L;
        size = 0;
        overflow = false;
        for (int j = 0; j < segments.length; ++j) {
            Segment<K,V> seg = segmentAt(segments, j);
            if (seg != null) {
                sum += seg.modCount;
                int c = seg.count;
                if (c < 0 || (size += c) < 0)
                    overflow = true;
            }
        }
        if (sum == last)
            break;
        last = sum;
    }
} finally {
    if (retries > RETRIES_BEFORE_LOCK) {
        for (int j = 0; j < segments.length; ++j)
            segmentAt(segments, j).unlock();
    }
}
```

先采用不加锁的方式，连续计算元素的个数，最多计算3次：
1、如果前后两次计算结果相同，则说明计算出来的元素个数是准确的；
2、如果前后两次计算结果都不同，则给每个`Segment`进行加锁，再计算一次元素的个数；

### 1.8实现

#### 数据结构

1.8中放弃了`Segment`臃肿的设计，取而代之的是采用`Node` + `CAS` + `Synchronized`来保证并发安全进行实现，结构如下：

![img](http://upload-images.jianshu.io/upload_images/2184951-d9933a0302f72d47.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

只有在执行第一次`put`方法时才会调用`initTable()`初始化`Node`数组，实现如下：

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

#### put实现

当执行`put`方法插入数据时，根据key的hash值，在`Node`数组中找到相应的位置，实现如下：

1、如果相应位置的`Node`还未初始化，则通过CAS插入相应的数据；

```java
else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
    if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
        break;                   // no lock when adding to empty bin
}
```

2、如果相应位置的`Node`不为空，且当前该节点不处于移动状态，则对该节点加`synchronized`锁，如果该节点的`hash`不小于0，则遍历链表更新节点或插入新节点；

```java
if (fh >= 0) {
    binCount = 1;
    for (Node<K,V> e = f;; ++binCount) {
        K ek;
        if (e.hash == hash &&
            ((ek = e.key) == key ||
             (ek != null && key.equals(ek)))) {
            oldVal = e.val;
            if (!onlyIfAbsent)
                e.val = value;
            break;
        }
        Node<K,V> pred = e;
        if ((e = e.next) == null) {
            pred.next = new Node<K,V>(hash, key, value, null);
            break;
        }
    }
}
```

3、如果该节点是`TreeBin`类型的节点，说明是红黑树结构，则通过`putTreeVal`方法往红黑树中插入节点；

```java
else if (f instanceof TreeBin) {
    Node<K,V> p;
    binCount = 2;
    if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key, value)) != null) {
        oldVal = p.val;
        if (!onlyIfAbsent)
            p.val = value;
    }
}
```

4、如果`binCount`不为0，说明`put`操作对数据产生了影响，如果当前链表的个数达到8个，则通过`treeifyBin`方法转化为红黑树，如果`oldVal`不为空，说明是一次更新操作，没有对元素个数产生影响，则直接返回旧值；

```java
if (binCount != 0) {
    if (binCount >= TREEIFY_THRESHOLD)
        treeifyBin(tab, i);
    if (oldVal != null)
        return oldVal;
    break;
}
```

5、如果插入的是一个新节点，则执行`addCount()`方法尝试更新元素个数`baseCount`；

#### size实现

1.8中使用一个`volatile`类型的变量`baseCount`记录元素的个数，当插入新数据或则删除数据时，会通过`addCount()`方法更新`baseCount`，实现如下：

```java
if ((as = counterCells) != null ||
    !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
    CounterCell a; long v; int m;
    boolean uncontended = true;
    if (as == null || (m = as.length - 1) < 0 ||
        (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
        !(uncontended =
          U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
        fullAddCount(x, uncontended);
        return;
    }
    if (check <= 1)
        return;
    s = sumCount();
}
```

1、初始化时`counterCells`为空，在并发量很高时，如果存在两个线程同时执行`CAS`修改`baseCount`值，则失败的线程会继续执行方法体中的逻辑，使用`CounterCell`记录元素个数的变化；

2、如果`CounterCell`数组`counterCells`为空，调用`fullAddCount()`方法进行初始化，并插入对应的记录数，通过`CAS`设置cellsBusy字段，只有设置成功的线程才能初始化`CounterCell`数组，实现如下：

```java
else if (cellsBusy == 0 && counterCells == as &&
         U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
    boolean init = false;
    try {                           // Initialize table
        if (counterCells == as) {
            CounterCell[] rs = new CounterCell[2];
            rs[h & 1] = new CounterCell(x);
            counterCells = rs;
            init = true;
        }
    } finally {
        cellsBusy = 0;
    }
    if (init)
        break;
}
```

3、如果通过`CAS`设置cellsBusy字段失败的话，则继续尝试通过`CAS`修改`baseCount`字段，如果修改`baseCount`字段成功的话，就退出循环，否则继续循环插入`CounterCell`对象；

```java
else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
    break;
```

所以在1.8中的`size`实现比1.7简单多，因为元素个数保存`baseCount`中，部分元素的变化个数保存在`CounterCell`数组中，实现如下：

```java
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 :
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
            (int)n);
}

final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```

通过累加`baseCount`和`CounterCell`数组中的数量，即可得到元素的总个数；



### 一致性

为什么JDK1.6的是弱一致性的？因为 JDK1.6 的所有可见性都是以 count 实现的，当 put和 get 并发时，get 可能获取不到最新的结果，这就是JDK1.6中ConcurrentHashMap弱一致性问题，而 JDK1.7 的实现，对于每一个操作都是 Volatile 变量的操作，能保证线程之间的可见性，所以不存在弱一致性的问题。

### 总结

ConcurrentHashMap 的高并发性主要来自于三个方面：
用分离锁实现多个线程间的更深层次的共享访问。
用 HashEntery 对象的不变性来降低执行读操作的线程在遍历链表期间对加锁的需求。
通过对同一个 Volatile 变量的写 / 读访问，协调不同线程间读 / 写操作的内存可见性。



ConcurrentHashMap 是一个并发散列映射表的实现，使用了分段锁的设计，【分段锁】在 hashMap 的基础上，将数据分为多个 segment（默认16，也可在申明时自己设置，不过一旦设定就不能更改，扩容都是扩充各个segment 的容量），每个 segment 就类似一个 HashTable，但比 HashTable 更加优化。每次操作对一个 segment 加锁，加锁操作是针对 hash 值对应的某个 Segment，而不是整个 ConcurrentHashMap。只有在每个段上才会存在并发争用，不同的段之间不存在锁的竞争，提高了该 hashmap 的并发访问效率。只要多个线程访问的不是同一个 segment 就没有锁争用，就没有堵塞，也就是允许16个线程并发的更新而尽量没有锁争用。使用分离锁，通过减小请求同一个锁的频率和尽量减少持有锁的时间 ，使得 ConcurrentHashMap 的并发性相对于 HashTable 和用同步包装器包装的 HashMap有了质的提高。

【读】在实际的应用中，散列表一般的应用场景是：除了少数插入操作和删除操作外，绝大多数都是读取操作。HashTable 对 get，put，remove 方法都会使用锁，而ConcurrnetHashMap 中get 方法是不涉及到锁的，因为 HashEntry 中 value 是 volatile 类型，通过 HashEntery 对象的不变性及对同一个 Volatile 变量的读 / 写来协调内存可见性，使得读操作大多数时候不需要加锁就能成功获取到需要的值。但需要注意的是，这也导致了 ConcurrentHashMap 的弱一致性问题，在对 get 操作有强一致性要求的场景下，必须选用其它支持map强一致性的容器。



[深入分析ConcurrentHashMap1.8](http://www.jianshu.com/p/f6730d5784ad)

[深入理解ConcurrentHashmap（JDK1.6到1.7）](http://www.jianshu.com/p/bd972088a494)
