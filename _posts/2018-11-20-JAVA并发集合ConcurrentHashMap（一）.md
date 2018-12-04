---
layout: post
title: JAVA 并发集合 ConcurrentHashMap（一）
categories: Java并发
description: JAVA 并发集合 ConcurrentHashMap（一）
keywords: Java, ConcurrentHashMap
---

今天收到一张 JAVA 并发体系的图，颇有兴趣，遂又开一个坑，慢慢写写这里面的东西。

本文基于 jdk1.8 。

# ConcurrentHashMap

CAS + Synchronized 来保证并发更新的安全，底层采用数组 + 链表 / 红黑树的存储结构

## 重要内部类

### Node

> Node 是 ConcurrentHashMap 存储结构的基本单元，用于存储 key-value 键值对，是一个链表，但只允许查找数据，不允许修改数据。源码如下：

```
/**
 * Key-value entry.  This class is never exported out as a
 * user-mutable Map.Entry (i.e., one supporting setValue; see
 * MapEntry below), but can be used for read-only traversals used
 * in bulk tasks.  Subclasses of Node with a negative hash field
 * are special, and contain null keys and values (but are never
 * exported).  Otherwise, keys and vals are never null.
 */
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    // value和next使用volatile来保证可见性和禁止重排序
    volatile V val;
    volatile Node<K,V> next;  // 指向下一个Node节点
 
    Node(int hash, K key, V val, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.val = val;
        this.next = next;
    }
 
    public final K getKey()       { return key; }
    public final V getValue()     { return val; }
    public final int hashCode()   { return key.hashCode() ^ val.hashCode(); }
    public final String toString(){ return key + "=" + val; }
    // 不允许更新value
    public final V setValue(V value) {
        throw new UnsupportedOperationException();
    }
 
    /**
     * Virtualized support for map.get(); overridden in subclasses.
     */
    // 用于map.get()方法，子类重写
    Node<K,V> find(int h, Object k) {
        Node<K,V> e = this;
        if (k != null) {
            do {
                K ek;
                if (e.hash == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
            } while ((e = e.next) != null);
        }
        return null;
    }
}
```

### TreeNode

> 红黑树节点，这里是遍历红黑树的源码

```
/**
 * Returns the TreeNode (or null if not found) for the given key
 * starting at given root.
 * 通过hash值的比较，递归的去遍历红黑树，这里要提的是compareableClassFor(Class k)这个函数的作用，在某些时候如果红黑树节点的元素are of the same "class C implements Comparable<C>" type利用他们的compareTo()方法来比较大小，这里需要通过反射机制来check他们到底是不是属于同一个类,是不是具有可比较性.
 * 寻找对象的过程是典型的先序遍历。
 */
final TreeNode<K,V> findTreeNode(int h, Object k, Class<?> kc) {
    if (k != null) {
        TreeNode<K,V> p = this;
        do  {
            int ph, dir; K pk; TreeNode<K,V> q;
            TreeNode<K,V> pl = p.left, pr = p.right;
            if ((ph = p.hash) > h)
                p = pl;
            else if (ph < h)
                p = pr;
            else if ((pk = p.key) == k || (pk != null && k.equals(pk)))
                return p;
            else if (pl == null)
                p = pr;
            else if (pr == null)
                p = pl;
            else if ((kc != null ||
                      (kc = comparableClassFor(k)) != null) &&
                     (dir = compareComparables(kc, k, pk)) != 0)
                p = (dir < 0) ? pl : pr;
            else if ((q = pr.findTreeNode(h, k, kc)) != null)
                return q;
            else
                p = pl;
        } while (p != null);
    }
    return null;
}
```

### TreeBin

> 相当于一颗红黑树，其构造方法其实就是构造红黑树的过程，封装 TreeNode 的容器，提供了转化红黑树的一些条件和锁的控制，封装了红黑树增删以及旋转节点操作，并且存了树的根节点

### ForwardingNode

> 辅助节点，用于 ConcurrentHashMap扩容操作，只有 table 发生扩容的时候，ForwardingNode 才会发挥作用，作为一个占位符放在 table 中表示当前节点为 null 或则已经被移动。

关于线程不安全的 HashMap 的扩容会出现死循环的情况，这个文章 [A Beautiful Race Condition](https://mailinator.blogspot.com/2009/06/beautiful-race-condition.html) 讲的很棒。

### sizeCtr

> 控制标识符，用来控制table初始化和扩容操作的

* 负数代表正在初始化或扩容操作

* -1 代表正在初始化

* -N 表示有 N-1 个线程正在进行扩容操作

* 正数或 0 代表 hash 表还没有被初始化，这个数值表示初始化或下一次扩容的大小。

## 重要操作

### initTable

ConcurrentHashMap 初始化方法，这个操作不在构造函数调用，而是在 put 的时候调用。

执行put操作的线程会执行 Unsafe.compareAndSwapInt 方法修改 sizeCtl 为 -1，有且只有一个线程能够修改成功。其它线程通过Thread.yield()让出CPU时间片等待table初始化完成。

这里就是之前 JAVA 锁中讲到的 CAS 了：

> CAS 比较交换的过程可以通俗的理解为 CAS(V,O,N) ，包含三个值分别为：V 内存地址存放的实际值；O 预期的值（旧值）；N 更新的新值。当 V 和 O 相同时，也就是说旧值和内存中实际的值相同表明该值没有被其他线程更改过，即该旧值 O 就是目前来说最新的值了，自然而然可以将新值N赋值给 V 。反之， V 和 O 不相同，表明该值已经被其他线程改过了则该旧值 O 不是最新版本的值了，所以不能将新值 N 赋给 V ，返回 V 即可。当多个线程使用 CAS 操作一个变量是，只有一个线程会成功，并成功更新，其余会失败。失败的线程会重新尝试，当然也可以选择挂起线程。

### put

table 为 null，线程进入初始化步骤，如果有其他线程正在初始化，该线程挂起。

如果插入的当前 i 位置为 null，说明该位置是第一次插入，利用 CAS 插入节点即可，插入成功，则调用 addCount 判断是否需要哦扩容。插入失败，自旋。

若该节点的 hash == MOVED（-1） ，表示有线程正在进行扩容，则进入扩容进程。

其余情况就是按照链表或者红黑树结构插入节点，需要同步（加锁）。

### get

从链表/红黑树节点获取

### 扩容

构建一个 nextTable，其大小为原来的两倍，这个步骤也是 CAS 操作

对于get读操作，如果当前节点有数据，还没迁移完成，此时不影响读，能够正常进行。

如果当前链表已经迁移完成，那么头节点会被设置成fwd节点，此时get线程会帮助扩容。（多线程操作）

所在链表的元素个数达到了阀值 8，则将链表转换为红黑树

## 1.8 和 1.7 的区别

红黑树和多线程扩容




