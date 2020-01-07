# ConcurrentLinkedQueue（**非阻塞算法**）



## 简介

一个基于链接节点的无界线程安全队列。此队列按照 FIFO（先进先出）原则对元素进行排序。队列的头部 是队列中时间最长的元素。队列的尾部 是队列中时间最短的元素。
新的元素插入到队列的尾部，队列获取操作从队列头部获得元素。当多个线程共享访问一个公共 collection 时，ConcurrentLinkedQueue 是一个恰当的选择。此队列不允许使用 null 元素。



## 定义

一个基于<b>链接节点的无界线程安全队列</b>。此队列按照 FIFO（先进先出）原则对元素进行排序。当我们添加一个元素的时候，它会添加到队列的尾部，当我们获取一个元素时，它会返回队列头部的元素。它采用了“wait－free”算法来实现，该算法在Michael & Scott算法上进行了一些修改。此队列不允许使用 null 元素。

## 不变式

队列有时会处于不一致状态。为此，ConcurrentLinkedQueue 使用三个不变式 ( 基本不变式，head 的不变式和 tail 的不变式 )，来约束队列中方法的执行。通过这三个不变式来维护非阻塞算法的正确性。

### 基本不变式

在执行方法之前和之后，队列必须要保持的不变式：

- 当入队插入新节点之后，队列中有一个 next 域为 null 的（最后）节点。
- 从 head 开始遍历队列，可以访问所有 item 域不为 null 的节点。

### head 的不变式和可变式

在执行方法之前和之后，head 必须保持的不变式：

- 所有“活着”的节点（指未删除节点），都能从 head 通过调用 succ() 方法遍历可达。
- head 不能为 null。
- head 节点的 next 域不能引用到自身。

在执行方法之前和之后，head 的可变式：

- head 节点的 item 域可能为 null，也可能不为 null。
- 允许 tail 滞后（lag behind）于 head，也就是说：从 head 开始遍历队列，不一定能到达 tail。

### tail 的不变式和可变式

在执行方法之前和之后，tail 必须保持的不变式：

- 通过 tail 调用 succ() 方法，最后节点总是可达的。
- tail 不能为 null。

在执行方法之前和之后，tail 的可变式：

- tail 节点的 item 域可能为 null，也可能不为 null。
- 允许 tail 滞后于 head，也就是说：从 head 开始遍历队列，不一定能到达 tail。
- tail 节点的 next 域可以引用到自身。

在接下来的源代码分析中，在初始化 ConcurrentLinkedQueue 之后及调用入队 / 出队方法之前和之后，我们都会参照上面三个不变式来分析它们的正确性。



## 初始化状态结构图

当初始化一个 ConcurrentLinkedQueue 对象时，会创建一个 item 域为 null 和 next 域为 null 的伪节点，并让 head 和 tail 指向这个伪节点，处于初始化状态的队列满足三个不变式。下面是队列初始化之后的结构示意图：

![初始化状态结构图](https://tva1.sinaimg.cn/large/006tNbRwgy1gaj58870yoj307i057q2x.jpg)

## 入队操作

```java
// 将指定元素插入此队列的尾部
public boolean offer(E e) {
    // 创建入队节点，如果e为null，则直接抛出NullPointerException异常
    final Node<E> newNode = new Node<E>(Objects.requireNonNull(e));
	// 循环CAS直到入队成功
    // 1、根据tail节点定位出尾节点（last node）；
    // 2、将新节点置为尾节点的下一个节点；
    // 3、casTail更新尾节点。
    for (Node<E> t = tail, p = t;;) {
        // p用来表示队列的尾节点，初始情况下等于tail节点
        // q是p的next节点
        Node<E> q = p.next;
        // 判断p是不是尾节点，tail节点不一定是尾节点，判断是不是尾节点的依据是该节点的next是不是null
        if (q == null) {// 如果p是尾节点
            // p is last node
            // 设置p节点的下一个节点为新节点，设置成功则casNext返回true；否则返回false，说明有其他线程更新过尾节点
            if (NEXT.compareAndSet(p, null, newNode)) {
                // Successful CAS is the linearization point
                // for e to become an element of this queue,
                // and for newNode to become "live".
                // 如果p != t，则将入队节点设置成tail节点，更新失败了也没关系，因为失败了表示有其他线程成功更新了tail节点
                if (p != t) // hop two nodes at a time; failure is OK
                    TAIL.weakCompareAndSet(this, t, newNode);
                return true;
            }
            // Lost CAS race to another thread; re-read next
        }
        // 多线程操作时候，由于poll时候会把旧的head变为自引用，然后将head的next设置为新的head
        // 所以这里需要重新找新的head，因为新的head后面的节点才是激活的节点
        else if (p == q)
            // We have fallen off list.  If tail is unchanged, it
            // will also be off-list, in which case we need to
            // jump to head, from which all live nodes are always
            // reachable.  Else the new tail is a better bet.
            p = (t != (t = tail)) ? t : head;
        // 寻找尾节点
        else
            // Check for tail updates after two hops.
            p = (p != t && t != (t = tail)) ? t : q;
    }
}
```



## 出队操作

```java
// 获取并移除此队列的头，如果此队列为空，则返回 null
public E poll() {
    restartFromHead: for (;;) {
        // p节点表示首节点，即需要出队的节点
        for (Node<E> h = head, p = h, q;; p = q) {
            final E item;
            // 如果p节点的元素item不为null，则通过CAS来设置p节点引用的元素为null，如果成功则返回p节点的元素
            if ((item = p.item) != null && p.casItem(item, null)) {
                // Successful CAS is the linearization point
                // for item to be removed from this queue.
                if (p != h) // hop two nodes at a time
                    updateHead(h, ((q = p.next) != null) ? q : p);
                return item;
            }
            // 如果头节点的元素为空或头节点发生了变化，这说明头节点已经被另外一个线程修改了。
            // 那么获取p节点的下一个节点，如果p节点的下一节点为null，则表明队列已经空了
            else if ((q = p.next) == null) {
                updateHead(h, p);
                return null;
            }
            // p == q，则使用新的head重新开始
            else if (p == q)
                continue restartFromHead;
        }
    }
}
```





## 总结

ConcurrentLinkedQueue 的非阻塞算法实现可概括为下面 5 点：

- 使用 CAS 原子指令来处理对数据的并发访问，这是非阻塞算法得以实现的基础。\
- head/tail 并非总是指向队列的头 / 尾节点，也就是说允许队列处于不一致状态。 这个特性把入队 / 出队时，原本需要一起原子化执行的两个步骤分离开来，从而缩小了入队 / 出队时需要原子化更新值的范围到唯一变量。这是非阻塞算法得以实现的关键。
- 由于队列有时会处于不一致状态。为此，ConcurrentLinkedQueue 使用三个不变式来维护非阻塞算法的正确性。
- 以批处理方式来更新 head/tail，从整体上减少入队 / 出队操作的开销。
- 为了有利于垃圾收集，队列使用特有的 head 更新机制；为了确保从已删除节点向后遍历，可到达所有的非删除节点，队列使用了特有的向后推进策略。