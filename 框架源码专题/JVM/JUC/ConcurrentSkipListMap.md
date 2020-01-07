# ConcurrentSkipListMap

## 介绍

ConcurrentSkipListMap是线程安全的有序的哈希表，适用于高并发的场景。
ConcurrentSkipListMap和TreeMap，它们虽然都是有序的哈希表。但是，第一，它们的线程安全机制不同，TreeMap是非线程安全的，而ConcurrentSkipListMap是线程安全的。第二，ConcurrentSkipListMap是通过<font color=red>跳表</font>实现的，而TreeMap是通过<font color=red>红黑树</font>实现的。


https://www.cnblogs.com/java-zzl/p/9767255.html


![跳表](https://tva1.sinaimg.cn/large/006tNbRwgy1gamw4gasq6j30th0fc0u3.jpg)