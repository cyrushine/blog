---
title: JUC 下一些线程安全的容器
date: 2021-03-31 12:00:00 +0800
categories: [JDK, JUC]
tags: [JUC, 线程安全]
---

## 写时复制（Copy On Write）

### `CopyOnWriteArrayList`

使用 **写时复制** 实现的线程安全版 `ArrayList`，当发生修改操作时（add、set、remove）才加锁，将原数组复制一份并在上面修改成为新数组，最后用新数组替换原数组

```java
public boolean add(E e) {
    synchronized (lock) {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    }
}
```

也就是说所有的修改操作都不会修改原数组，这样所有的读操作（get、iterate）都可以不加锁，从而实现高效的读（虽然有可能会读到旧数据）；因为它的写操作是很昂贵的（复制一份出来），但同时它的读操作和迭代很高效（不上锁），所以它适用于读操作远大于写操作的情况；`CopyOnWriteArraySet` 内部是通过 `CopyOnWriteArrayList` 实现的

## 分段加锁

### `ConcurrentHashMap`

跟 `HashMap` 一样采用数组 + 链表的实现，链表又叫做桶 or 箱子（bin）

```java
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
    implements ConcurrentMap<K,V>, Serializable {
    transient volatile Node<K,V>[] table;
}
```

写时采用 **分段加锁**，不对整个写操作 or `table` 加锁，而只对所在的桶加锁，其他线程依然可以进行读操作 or 对其他桶进行写操作

整个 `ConcurrentHashMap` 都没有使用 `Lock` 进行阻塞，而是尽可能采用自旋 + CAS（乐观锁，是实现无锁操作的重要函数），最后才用 `synchronized`（参考文章，它的锁膨胀过程中掺杂自旋和阻塞）对桶上锁

`tabAt`、`setTabAt` 和 `casTabAt` 使对 `table` 的操作具有可见性和原子性，避免了对 `table` 上锁

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // 没有对整个写操作加锁，也没有对 table 加锁
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;

    // CAS 失败会自旋，是乐观锁
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;

        // 如果 table == null，则进行初始化；初始化后每个桶是 null
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        
        // 所在的桶为 null，不加锁直接用 CAS 操作添加新桶头，失败的话自旋
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {

            // 找到所在的不为 null 的桶，对单个桶上锁
            V oldVal = null;
            synchronized (f) {
                if (tabAt(tab, i) == f) {

                    // 沿着链表从头开始走，如果找到 key 值相等的节点则覆盖旧的 value
                    // 否则作为新节点添加到链表尾部
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
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    // ... 链表被树化为红黑树的情况参考 HashMap 的文章
                }
            }
            // ...
        }
    }
    // ...
}

// 初始化 table，自旋 + CAS
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

// 使数组的读/写操作像 volatile 成员变量一样具有线程可见性
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}
static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
    U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
}

// 在数组上实现 CAS 操作
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                    Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
```

虽然都是线程安全的 map，但 `ConcurrentHashMap` 的分段加锁对比 `HashTable` 的整个方法加锁优势就体现出来了，高并发下优势会愈加明显

```java
// HashTable 对整个写操作加锁
public synchronized V put(K key, V value) {
    // Make sure the value is not null
    if (value == null) {
        throw new NullPointerException();
    }

    // Makes sure the key is not already in the hashtable.
    HashtableEntry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;
    @SuppressWarnings("unchecked")
    HashtableEntry<K,V> entry = (HashtableEntry<K,V>)tab[index];
    for(; entry != null ; entry = entry.next) {
        if ((entry.hash == hash) && entry.key.equals(key)) {
            V old = entry.value;
            entry.value = value;
            return old;
        }
    }

    addEntry(hash, key, value, index);
    return null;
}
```

读操作完全不加锁，但是 `Node.val` 和 `Node.next` 是 `volatile` 修饰的，所以 `Node` 的线程可见性是有保证的

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}

static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;
    volatile Node<K,V> next;
}
```

## `BlockingQueue`

它提供的阻塞操作包括：
|                                    |                         |
|------------------------------------|-------------------------|
| `put(e)`                           | 入队                     |
| `offer(e, timeout, unit)`          | 设置超时的入队             |
| `take()`                           | 出队                     |
| `poll(timeout, unit)`              | 设置超时的出队             |
| `drainTo(collection, maxElements)` | 批量出队并添加到另一个集合中 |

### `ArrayBlockingQueue`

基于数组、容量有限的阻塞队列，通过构造函数指定队列的容量

用两个指针 `takeIndex`(队头，指向下一次出队的位置) 和 `putIndex`（队尾，指向下一次入队的位置） 模拟队列，它们初始为 0（最左边），随着元素的入队 `putIndex` 往右移动，随着元素的出队 `takeIndex` 也往右移动，当它们越过数组最后边时会重置到最左边，`count` 确保 `takeIndex` 不会违规越过 `putIndex`

```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {

    /** The queued items */
    final Object[] items;

    /** items index for next take, poll, peek or remove */
    int takeIndex;

    /** items index for next put, offer, or add */
    int putIndex;

    /** Number of elements in the queue */
    int count;

}
```

出队/入队时用了 `Lock` 和 `Condition` 实现阻塞和唤醒，出队时如果为空则阻塞在 `notEmpty` 上，入队时如果满了则阻塞在 `notFull`，入队后唤醒阻塞在 `notEmpty` 上的线程，出队后唤醒阻塞在 `notFull` 上的线程

```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {

    /** Main lock guarding all access */
    final ReentrantLock lock;

    /** Condition for waiting takes */
    private final Condition notEmpty;

    /** Condition for waiting puts */
    private final Condition notFull;

    /**
     * Inserts element at current put position, advances, and signals.
     * Call only when holding lock.
     */
    private void enqueue(E x) {
        // assert lock.getHoldCount() == 1;
        // assert items[putIndex] == null;
        final Object[] items = this.items;
        items[putIndex] = x;
        if (++putIndex == items.length) putIndex = 0;
        count++;
        notEmpty.signal();
    }

    /**
     * Extracts element at current take position, advances, and signals.
     * Call only when holding lock.
     */
    private E dequeue() {
        // assert lock.getHoldCount() == 1;
        // assert items[takeIndex] != null;
        final Object[] items = this.items;
        @SuppressWarnings("unchecked")
        E x = (E) items[takeIndex];
        items[takeIndex] = null;
        if (++takeIndex == items.length) takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
        notFull.signal();
        return x;
    }  

}
```

### `LinkedBlockingQueue`

基于链表、无限容量（当然也可以通过构造函数设置最大容量）的阻塞队列，链表是单向的，`head` 指向队头也就是出队的位置，`last` 指向队尾也就是出队的位置

```java
public class LinkedBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {

    /**
     * Linked list node class.
     */
    static class Node<E> {
        E item;

        /**
         * One of:
         * - the real successor Node
         * - this Node, meaning the successor is head.next
         * - null, meaning there is no successor (this is the last node)
         */
        Node<E> next;

        Node(E x) { item = x; }
    }

    /** The capacity bound, or Integer.MAX_VALUE if none */
    private final int capacity;

    /** Current number of elements */
    private final AtomicInteger count = new AtomicInteger();

    /**
     * Head of linked list.
     * Invariant: head.item == null
     */
    transient Node<E> head;

    /**
     * Tail of linked list.
     * Invariant: last.next == null
     */
    private transient Node<E> last;

    private void enqueue(Node<E> node) {
        // assert putLock.isHeldByCurrentThread();
        // assert last.next == null;
        last = last.next = node;
    }

    private E dequeue() {
        // assert takeLock.isHeldByCurrentThread();
        // assert head.item == null;
        Node<E> h = head;
        Node<E> first = h.next;
        h.next = h; // help GC
        head = first;
        E x = first.item;
        first.item = null;
        return x;
    }

}
```

跟 `ArrayBlockingQueue` 一样用了两个条件变量：`notEmpty` 和 `notFull` 来阻塞/唤醒生产者和消费者；为啥会有 `notFull` 的情况呢，不是无限容量吗？因为它可以设置一个最大容量

不同的是 `LinkedBlockingQueue` 用了两个锁，`takeLock` 给出队加锁，`putLock` 给入队加锁，出队和入队之所以可以并行是有 `count` 在确保数量正确

```java
public class LinkedBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {

    /** The capacity bound, or Integer.MAX_VALUE if none */
    private final int capacity;

    /** Current number of elements */
    private final AtomicInteger count = new AtomicInteger();

    /** Lock held by take, poll, etc */
    private final ReentrantLock takeLock = new ReentrantLock();

    /** Wait queue for waiting takes */
    private final Condition notEmpty = takeLock.newCondition();

    /** Lock held by put, offer, etc */
    private final ReentrantLock putLock = new ReentrantLock();

    /** Wait queue for waiting puts */
    private final Condition notFull = putLock.newCondition();

}
```

### `PriorityBlockingQueue`

上面两个阻塞队列是 `FIFO` 排序的，而这个可以用 `Comparator` 和 `Comparable` 自定义优先级

底层用 **小顶堆** 实现的优先队列，小顶堆是用数组实现的二叉树（左右节点要大于父节点）；入队元素添加到叶子那层的最左边，然后自下往上跟父节点比较，如果小则交换，这个操作叫 `siftUp`；出队元素固定是树的根节点，出队后把最后一个节点作为根节点，从上往下跟左右节点比较，如果大则交换，这个操作叫 `siftDown`（参考 <a href="../threadpool/">这篇文章</a> 里堆的介绍）

```java
public class PriorityBlockingQueue<E> extends AbstractQueue<E>
    implements BlockingQueue<E>, java.io.Serializable {

    /**
     * Priority queue represented as a balanced binary heap: the two
     * children of queue[n] are queue[2*n+1] and queue[2*(n+1)].  The
     * priority queue is ordered by comparator, or by the elements'
     * natural ordering, if comparator is null: For each node n in the
     * heap and each descendant d of n, n <= d.  The element with the
     * lowest value is in queue[0], assuming the queue is nonempty.
     */
    private transient Object[] queue;

    private static <T> void siftUpUsingComparator(int k, T x, Object[] array,
                                       Comparator<? super T> cmp) {
        while (k > 0) {
            int parent = (k - 1) >>> 1;
            Object e = array[parent];
            if (cmp.compare(x, (T) e) >= 0)
                break;
            array[k] = e;
            k = parent;
        }
        array[k] = x;
    }

    private static <T> void siftDownUsingComparator(int k, T x, Object[] array,
                                                    int n,
                                                    Comparator<? super T> cmp) {
        if (n > 0) {
            int half = n >>> 1;
            while (k < half) {
                int child = (k << 1) + 1;
                Object c = array[child];
                int right = child + 1;
                if (right < n && cmp.compare((T) c, (T) array[right]) > 0)
                    c = array[child = right];
                if (cmp.compare(x, (T) c) <= 0)
                    break;
                array[k] = c;
                k = child;
            }
            array[k] = x;
        }
    }
}
```

虽然底层是数组但可以扩容，也即无限容量；扩容操作也很细致地分为两步：
* 分配一块新内存，用 int 和 CAS 操作实现自旋（也许是认为分配新内存很快，所以用乐观锁？）
* 复制数组用 `Lock`

```java
private void tryGrow(Object[] array, int oldCap) {
    // 分配内存，自旋
    lock.unlock(); // must release and then re-acquire main lock
    Object[] newArray = null;
    if (allocationSpinLock == 0 &&
        U.compareAndSwapInt(this, ALLOCATIONSPINLOCK, 0, 1)) {
        try {
            int newCap = oldCap + ((oldCap < 64) ?
                                   (oldCap + 2) : // grow faster if small
                                   (oldCap >> 1));
            if (newCap - MAX_ARRAY_SIZE > 0) {    // possible overflow
                int minCap = oldCap + 1;
                if (minCap < 0 || minCap > MAX_ARRAY_SIZE)
                    throw new OutOfMemoryError();
                newCap = MAX_ARRAY_SIZE;
            }
            if (newCap > oldCap && queue == array)
                newArray = new Object[newCap];
        } finally {
            allocationSpinLock = 0;
        }
    }
    if (newArray == null) // back off if another thread is allocating
        Thread.yield();

    // 复制数组才上悲观锁
    lock.lock();
    if (newArray != null && queue == array) {
        queue = newArray;
        System.arraycopy(array, 0, newArray, 0, oldCap);
    }
}
```

跟 `ArrayBlockingQueue` 一样，入队/出队用同一把锁，因为无容量限制所以只需一个条件变量 `notEmpty`（`notFull` 的情况不会出现）

```java
public class PriorityBlockingQueue<E> extends AbstractQueue<E>
    implements BlockingQueue<E>, java.io.Serializable {

    /**
     * Lock used for all public operations.
     */
    private final ReentrantLock lock;

    /**
     * Condition for blocking when empty.
     */
    private final Condition notEmpty;

}
```

### 总结比较

* 需要自定义优先级用 `PriorityBlockingQueue`，需要无限容量用 `LinkedBlockingQueue`
* `LinkedBlockingQueue` 入队出队分别使用两把锁，也就是说入队出队可以并行，在高并发下会比使用同一把锁的 `ArrayBlockingQueue` 性能要好
* `ArrayBlockingQueue` 在内存利用率上会比 `LinkedBlockingQueue` 要好（`Node` 需要额外的空间），而且底层数组在构造函数时就已预先分配内存，使用时无需动态申请内存，内存波动较小；而动态申请内存的 `LinkedBlockingQueue` 可能会增加 JVM GC 的负担