---
layout: post
title: JAVA 并发集合 ConcurrentLinkedQueue （二）。
---

ConcurrentLinkedQueue 是一个基于链接节点的无界线程安全队列，它采用先进先出的规则对节点进行排序，当我们添加一个元素的时候，它会添加到队列的尾部，当我们获取一个元素时，它会返回队列头部的元素。用CAS实现非阻塞的线程安全队列。 

本文基于 jdk1.8 。

# ConcurrentLinkedQueue

## 非阻塞算法

1. 使用 CAS 原子指令来处理对数据的并发访问，这是非阻塞算法得以实现的基础。 
2. head/tail 并非总是指向队列的头 / 尾节点，也就是说允许队列处于不一致状态。 这个特性把入队 / 出队时，原本需要一起原子化执行的两个步骤分离开来，从而缩小了入队 / 出队时需要原子化更新值的范围到唯一变量。这是非阻塞算法得以实现的关键。 
3. 以批处理方式来更新 head/tail，从整体上减少入队 / 出队操作的开销

## 基本不变性条件

1. 在入队时最后一个结点中的 next 域为 null
2. 队列中的所有未删除结点的 item 域不能为 null 且从 head 都可以在 O(N) 时间内遍历到
3. 对于要删除的结点，不是将其引用直接置为空，而是将其的 item 域先置为 null (迭代器在遍历是会跳过 item 为 null 的结点)
4. 允许 head 和 tail 滞后更新，也就是上文提到的 head/tail 并非总是指向队列的头 / 尾节点（这主要是为了减少 CAS 指令执行的次数，但同时会增加 volatile 读的次数，但是这种消耗较小）。具体而言就是，当在队列中插入一个元素是，会检测 tail 和最后一个结点之间的距离是否在两个结点及以上(内部称之为 hop )；而在出队时，对 head 的检测就是与队列的第一个结点的距离是否达到两个，有则将 head 指向第一个结点并将 head 原来指向的结点的 next 域指向自己，这样就能断开与队列的联系从而帮助 GC

## head的不变性和可变性条件

### 不变性：

1. 所有未删除节点，都能从 head 通过调用 succ() 方法遍历可达。

2. head 不能为 null。

3. head 节点的 next 域不能引用到自身。

### 可变性：

1. head节点的item域可能为null，也可能不为null。

2. 允许tail滞后（lag behind）于head，也就是说：从head开始遍历队列，不一定能到达tail。

## tail的不变性和可变性条件

### 不变性：

1. 通过tail调用succ()方法，最后节点总是可达的。

2. tail不能为null。

### 可变性：

1. tail节点的item域可能为null，也可能不为 null。

2. 允许tail滞后于head，也就是说：从head开始遍历队列，不一定能到达tail。

3. tail节点的next域可以引用到自身。

## 入队列

入队主要做两件事情，第一是将入队节点设置成当前队列尾节点的下一个节点。第二是更新tail节点，如果tail节点的next节点不为空，则将入队节点设置成tail节点，如果tail节点的next节点为空，则将入队节点设置成tail的next节点，所以tail节点不总是尾节点，


```
public boolean offer(E e) {
    checkNotNull(e);
    final Node<E> newNode = new Node<E>(e);
    // 不断循环尝试插入
    for (Node<E> t = tail, p = t;;) {
        Node<E> q = p.next;
        if (q == null) {
            // p is last node
            // 下一个节点为 null ，意味着 p 节点就是尾节点。
            if (p.casNext(null, newNode)) {
                // Successful CAS is the linearization point
                // for e to become an element of this queue,
                // and for newNode to become "live".
                // 将 newNode 设置为当前队列尾节点的 next 节点，
                if (p != t) // hop two nodes at a time
                // 判断 p 节点不是尾节点，更新一次。
                    casTail(t, newNode);  // Failure is OK.
                return true;
            }
            // Lost CAS race to another thread; re-read next
        }
        else if (p == q)
            // We have fallen off list.  If tail is unchanged, it
            // will also be off-list, in which case we need to
            // jump to head, from which all live nodes are always
            // reachable.  Else the new tail is a better bet.
            // 这里表示回环，需要重新从 head 寻找队列的结尾。
            p = (t != (t = tail)) ? t : head;
        else
            // Check for tail updates after two hops.
            // q 不是 null，说明 p 的 next 指向别的元素了，要从 q 开始循环找到最后一个元素
            // 利用上面的代码更新 tail 的位置
            p = (p != t && t != (t = tail)) ? t : q;
    }
}
```

这里的 else：
1. tail 指向自己，需要重新从 head 遍历
2. tail 指向非尾节点，即 tail 滞后

## 出队列

```
public E poll() {
    restartFromHead:
    for (;;) {
        for (Node<E> h = head, p = h, q;;) {
            E item = p.item;
            // 如果item不为null的话将其设为null实现删除头结点
            if (item != null && p.casItem(item, null)) {
                // Successful CAS is the linearization point
                // for item to be removed from this queue.
                if (p != h) // hop two nodes at a time
                    updateHead(h, ((q = p.next) != null) ? q : p);
                return item;
            }
            else if ((q = p.next) == null) {
                updateHead(h, p);
                return null;
            }
            else if (p == q)
                continue restartFromHead;
            else
                p = q;
        }
    }
}
```








