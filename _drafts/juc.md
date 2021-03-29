`CopyOnWriteArrayList`

使用 **写时复制** 实现的线程安全版 `ArrayList`
当发生修改操作时（add、set、remove）才加锁，将原数组复制一份并在上面修改成为新数组，最后用新数组替换原数组

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

也就是说所有的修改操作都不会修改原数组，这样所有的读操作（get、iterate）都可以不加锁，从而实现高效的读（虽然有可能会读到旧数据）

因为它的写操作是很昂贵的（复制一份出来），但同时它的读操作和迭代很高效（不上锁），所以它适用于读操作远大于写操作的情况

`CopyOnWriteArraySet` 内部是通过 `CopyOnWriteArrayList` 实现的

`ConcurrentHashMap`

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

