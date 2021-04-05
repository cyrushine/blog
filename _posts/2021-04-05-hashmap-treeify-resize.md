---
title: HashMap 的树化和扩容
date: 2021-04-05 12:00:00 +0800
categories: [JDK, Collection]
tags: [HashMap, 红黑树]
---

## HashMap 的桶由链表变为红黑树（树化）的过程

### 红黑树的特性

1. 节点为红色或者黑色
2. 根节点必须是黑的
3. 红色节点的左右子节点必须为黑色
4. 一个节点到叶子节点的每条路径必须包含相同数目的黑色节点

### 颜色变换和两种选择

添加新节点后，因为新节点总是红色的，那么会有几种情况出现：

1. 新节点是根节点，也就是说树是空的，根据规则二，把新节点设为黑色即可
2. 新节点的父节点是黑色，或者父节点是根，满足规则
3. 父节点是红色，违反规则三，需要进行 **平衡** 操作

平衡操作主要是根据情况组合使用下面三种转换（方块表示一棵满足红黑树规则的子树）：

![单旋](/assets/2021-04-05-hashmap-treeify-resize/op_1.png)

![双旋](/assets/2021-04-05-hashmap-treeify-resize/op_2.png)

![颜色变换](/assets/2021-04-05-hashmap-treeify-resize/op_3.png)

### 几个问题

#### 为什么要进行旋转？

由于 P（父节点 和 X（新节点）都为红色，违反规则三

#### 为什么新节点总是红色？

因为添加新节点前的树结构是构建好的，一但我们添加黑色节点，无论添加在哪里都会破坏原有路径上的黑色节点的数量平等关系，所以插入红色节点是正确的选择

#### 为什么要进行颜色变换？

如果叶子节点是红色的，那么我们在添加的时候只能添加黑色节点，然而添加任何黑色叶子节点都会违反规则四，所以要对其进行变换。进行变换后叶子节点是黑色的，而且我们默认添加的叶子节点是红色的，添加到红色的新节点后并不会违反规则四，所以这种变换很有用

#### 第二种双变换中在树的内部怎么出现的红色的节点？

正是由于上面的颜色变换导致颜色变换后的节点与他的父节点产生了颜色冲突

### HashMap 树化的过程

当满足下述条件时才将链表树化为红黑树

* 桶内元素超过 `TREEIFY_THRESHOLD = 8`（当桶内元素小于 `UNTREEIFY_THRESHOLD = 6` 时，红黑树会降级为链表）
* 桶的数量超过 `MIN_TREEIFY_CAPACITY`（小于这个数量只是进行扩容操作）；无论是链表还是红黑树，都是为了解决哈希冲突，如果桶太少则应该首先增加桶的数量降低哈希冲突出现的概率，其次才是用红黑树增加查找效率

首先将 `Node` 转变为 `TreeNode`，此时还是链表结构；第一个节点即为根，后面的节点作为新节点，按规则依次添加到树里：比父节点小则添加到左子树，比父节点大则添加到右子树，从上往下搜索直到要添加的子树为空即为新节点的位置；每次插入新节点后都需要进行 **平衡** 操作

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    // ... 将新元素添加至链表尾部（桶），如果桶的大小超过 TREEIFY_THRESHOLD，准备树化
    for (int binCount = 0; ; ++binCount) {
        if ((e = p.next) == null) {
            p.next = newNode(hash, key, value, null);
            if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                treeifyBin(tab, hash);
            break;
        }
    // ...
}

final void treeifyBin(Node<K,V>[] tab, int hash) {
    // 如果桶的数量 < MIN_TREEIFY_CAPACITY，只是扩容
    int n, index; Node<K,V> e;
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    
    // 桶多于 MIN_TREEIFY_CAPACITY 才树化
    // 将桶 tab[index] 里的节点转变为 TreeNode，但 此时还是链表
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            // hd 是链表头，从它开始树化
            hd.treeify(tab);
    }
}
```

```java
// 将还是链表的桶树化，当前是链表头
final void treeify(Node<K,V>[] tab) {

    // 链表里第一个元素作为红黑树初始的根
    // 遍历链表，逐个添加到红黑树中
    TreeNode<K,V> root = null;
    for (TreeNode<K,V> x = this, next; x != null; x = next) {
        next = (TreeNode<K,V>)x.next;
        x.left = x.right = null;
        if (root == null) {
            x.parent = null;
            x.red = false;
            root = x;
        }
        else {
            K k = x.key;
            int h = x.hash;
            Class<?> kc = null;

            // 从根开始，自上而下找位置
            // 比父节点小则插入到左子树，比父节点大则插入到右子树，直到所插入的位置为 null
            for (TreeNode<K,V> p = root;;) {

                // x - 新节点，p - 父节点，h - 新节点 hash，ph - parent hash
                // dir == -1，新节点比父节点小，添加到左子树；dir == 1，新节点比父节点大，添加到右子树
                int dir, ph;
                K pk = p.key;
                if ((ph = p.hash) > h)
                    dir = -1;
                else if (ph < h)
                    dir = 1;
                else if ((kc == null &&
                          (kc = comparableClassFor(k)) == null) ||
                         (dir = compareComparables(kc, k, pk)) == 0)
                    dir = tieBreakOrder(k, pk);

                // xp - 新节点的父节点
                // 一直找，直到新节点需要插入的位置是为 null，那么就把新节点放在那
                TreeNode<K,V> xp = p;
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    x.parent = xp;
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    // 插入新节点后可能会破坏红黑树的平衡，每次插入后都要执行平衡操作
                    root = balanceInsertion(root, x);
                    break;
                }
            }
        }
    }
    moveRootToFront(tab, root);
}
```

```java
// 插入新节点后需要平衡红黑树
static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root, TreeNode<K,V> x) {
    x.red = true; // 新节点总是红色的

    // xp - 新节点的 parent，xpp - 新节点的祖父，xppl - 祖父的左孩子，xppr - 祖父的右孩子
    for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {

        // 新节点没有父节点，说明它是根节点，根节点必须是黑色
        if ((xp = x.parent) == null) {
            x.red = false;
            return x;
        }

        // 1，不是根节点且父节点是黑色，满足红黑树的条件，返回即可
        // 2，祖父为 null 说明父节点为根，根一定是黑色，新节点为红色，满足条件
        else if (!xp.red || (xpp = xp.parent) == null)
            return root;

        // 父节点是红色且有祖父节点，那就比较麻烦了，必须要进行旋转和颜色变换操作，此时父节点是祖父的左孩子    
        if (xp == (xppl = xpp.left)) {

            // 新节点是红色，父节点也是红色，父节点旁边的兄弟节点也是红色（由于当前红黑树除新节点外是平衡的，所以祖父肯定是黑色）
            // 那么进行颜色变换：将父节点和它的兄弟节点变为黑色，祖父变为红色，对应图三
            // 祖父变色后，可能引起祖父上面不平衡，所以下次循环要操作祖父
            if ((xppr = xpp.right) != null && xppr.red) {
                xppr.red = false;
                xp.red = false;
                xpp.red = true;
                x = xpp;
            }
            else {

                // 父节点的兄弟为黑色 or 为空，那么就要通过旋转解决两个红色节点相连的问题
                // 新节点是右孩子，对应图二的双旋，这里是第一次的左旋
                if (x == xp.right) {
                    root = rotateLeft(root, x = xp);
                    xpp = (xp = x.parent) == null ? null : xp.parent;
                }

                // 继续上面的（左旋）后的第二次右旋
                // 或者对应图一的单旋（右旋）
                if (xp != null) {
                    xp.red = false;
                    if (xpp != null) {
                        xpp.red = true;
                        root = rotateRight(root, xpp);
                    }
                }
            }
        }

        // 同样是旋转和颜色变换操作，只不过父节点现在是祖父的右孩子，流程跟上面差不多的
        else {
            if (xppl != null && xppl.red) {
                xppl.red = false;
                xp.red = false;
                xpp.red = true;
                x = xpp;
            }
            else {
                if (x == xp.left) {
                    root = rotateRight(root, x = xp);
                    xpp = (xp = x.parent) == null ? null : xp.parent;
                }
                if (xp != null) {
                    xp.red = false;
                    if (xpp != null) {
                        xpp.red = true;
                        root = rotateLeft(root, xpp);
                    }
                }
            }
        }
    }
}
```

## 扩容的过程



## 参考

* (30张图带你彻底理解红黑树)[https://www.jianshu.com/p/e136ec79235c]