---
title: Java 源码研究之 HashMap
tags:
  - Java
  - HashMap
categories:
  - Java
date: 2018-01-13 13:09:00
---
本文是在观看 [Java HashMap 工作原理及实现](http://yikun.github.io/2015/04/01/Java-HashMap%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/) 后，虽然大致了解了 `HashMap` 的工作原理及实现，但是对实现的具体过程，思路尚未贯通，所以对于其中的几个核心方法按照每个步骤进行研究，注释

源码版本为`jdk1.8.0_91`

`put(K key, V value)`
```java
public V put(K key, V value) {
    // 调用 putVal 方法
    return putVal(hash(key), key, value, false, true);
}
// 对 key 进行 hash 操作
static final int hash(Object key) {
    int h;
    // 如果 key 为 null,返回0,否则调用 hashCode() 方法,然后对 hashCode 高16bit不变，低16bit和高16bit做了一个异或处理
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
`putVal(int hash, K key, V value, boolean onlyIfAbsent,boolean evict)`
```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 如果 table 为 null,或者 table 的长度为0,进行初始化操作
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 根据 (table长度-1) 与 hash 计算得出该 hash 在table中的索引
    // 根据索引获取对应的值,如果该值为 null,在此位置插入一个 Node 对象
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        // 判断该值的 hash,key 与要插入的 hash,key 是否相等
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            // 如果相等,表明该key为当前节点的第一个,将原值设置为当前 e 对象
            e = p;
        else if (p instanceof TreeNode)
            // 判断当前节点是否为 TreeNode 类型
            // 如果是 TreeNode 类型,使用红黑树的方式找出对应节点或新增节点并返回
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 如果是链表类型
            for (int binCount = 0; ; ++binCount) {
                // 如果下一个节点为 null,进行节点追加操作
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        // 如果当前节点的数量大于等于 8,将链表转换为 TreeNode
                        treeifyBin(tab, hash);
                    break;
                }
                // 如果链表中存在该 key,因为已经将该节点赋值给 e,所以直接结束循环,等待下面的方法对值进行更新
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // 如果 e 不等于 null,证明存在旧节点
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            // 更新原本旧值
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            // 空实现
            afterNodeAccess(e);
            return oldValue;
        }
    }
    // 操作数加1
    ++modCount;
    // 如果总数加1大于threshold,进行扩容
    if (++size > threshold)
        resize();
    // 空实现
    afterNodeInsertion(evict);
    return null;
}
```
`get(K key)`
```java
public V get(Object key) {
    Node<K,V> e;
    // 调用 getNode 方法进行获取 Node 对象
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```
`getNode(int hash, Object key)`
```java
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    // 如果 table 为 null 或 table 的长度为0 或 根据 hash 计算的节点为 null,返回null,否则进行查找
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        // 检查第一个节点是否为当前 key
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        // 如果第二个节点不是 null
        if ((e = first.next) != null) {
            // 如果是 TreeNode 类型
            if (first instanceof TreeNode)
                // 使用红黑树的查找方法进行查找
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            // 循环判断 hash,key是否相等,如果相等,返回否则一直到链表结束
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```
`remove(Object key)`
```java
public V remove(Object key) {
    Node<K,V> e;
    // 调用 removeNode 方法
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}
```
`removeNode(int hash, Object key, Object value,boolean matchValue, boolean movable)`
```java
final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    // 如果 table 为 null 或 table 的长度为0 或 根据 hash 计算的节点为 null,返回null,否则进行查找
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        // 检查第一个节点是否为当前 key,如果是将其赋值给 node 变量
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        // 如果第二个节点不为空
        else if ((e = p.next) != null) {
            // 如果是 TreeNode 类型
            if (p instanceof TreeNode)
                // 使用红黑树的查找方法进行查找
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                // 循环判断 hash,key 是否相等,如果相等,赋值给 node 变量
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        // 如果根据 key 找到对应节点
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            // 根据该节点的类型进行对应的删除操作
            if (node instanceof TreeNode)
                // 如果是 TreeNode 类型,按照红黑树的方式删除
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p)
                // 如果是第一个,将table中的该索引指向第二个节点
                tab[index] = node.next;
            else
                //如果是在链表中,将 node 的前一个节点的 next 指向 node 的节点的 next
                p.next = node.next;
            // 操作数加1
            ++modCount;
            // 总数减1
            --size;
            // 空实现
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```
`resize()`
```java
final Node<K,V>[] resize() {
    // 原数组
    Node<K,V>[] oldTab = table;
    // 原容量
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    // 原threshold值(容量*负载因子)
    int oldThr = threshold;
    int newCap, newThr = 0;
    // 如果原容量大于 0
    if (oldCap > 0) {
        // 如果数组长度达到最大上限,更新 threshold,不进行扩容
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 否则容量*2 threshold*2
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    // 如果在构造函数中设置了初始 threshold 使用 HashMap(int initialCapacity, float loadFactor)创建 Map
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
    // 如果原容量且原threshold 都为0,进行初始化操作
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 如果 newThr == 0 ( oldThr > 0 为 true 时该判断才会为 true)
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        // 计算新的 threshold
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    // 更新 threshold
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    // 根据newCap 构造一个新数组
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    // 更显table引用
    table = newTab;
    // 如果 oldTab 不为 null,表明为扩容操作,否则为table初始化操作
    if (oldTab != null) {
        // 遍历原数组中的元素
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            // 如果该元素不为 null
            if ((e = oldTab[j]) != null) {
                // 将原数组该索引设置为null,方便回收
                oldTab[j] = null;
                // 如果该节点下一个元素为null,表明该节点只存在一个元素
                if (e.next == null)
                    // 将该节点设置到新数组中去
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    // 如果节点为 TreeNode 类型,按照对应方式设置到新数组中
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    // 如果是数量大于1的链表
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        // 此处操作跟hash计算索引有关
                        // 在 HashMap 中,索引的计算方法为 (n - 1) & hash
                        // 所以,在进行扩容操作 (n*2) 后,该计算结果可能导致变更
                        // 例如
                        // 有一个值为 111001 的 hash
                        // 扩容前  n=16(10000)  n-1=15(1111)  (n - 1) & hash = 1111 & 111001= 001001
                        // 扩容后 n=32(100000) n-1=31(11111)  (n - 1) & hash = 11111 & 111001= 011001
                        // 假如 hash 值为 101001
                        // 那么会发现扩容前  1111 & 101001 = 001001
                        //           扩容后 11111 & 101001 = 001001
                        // 所以可知,在进行扩容操作时,主要按照 hash 与 原数组长度中1的对应位置有关
                        // 如果 hash 中对应的位置为0,扩容后索引结果不变
                        // 不为0,表示索引结果为原结果+原数组长度
                        // 而 hash 中该对应位置的值只存在俩种可能 0,1
                        // 所以在该节点中的数据大约有一半索引不变,一半为原索引+原数组长度
                        // 通过 e.hash & oldCap 的方式可以得知 hash 在 oldCap 1对应的位置是否为0或1
                        if ((e.hash & oldCap) == 0) {
                            // 如果为0,证明扩容后索引的计算依然与扩容前一致
                            // 组装链表
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            //如果不为0,则表明扩容后索引的计算依然与扩容不一致,所以需要移动到新索引,新索引的位置为旧索引加oldCap
                            // 组装链表
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 如果链表不为 null,设置到新数组中
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```



