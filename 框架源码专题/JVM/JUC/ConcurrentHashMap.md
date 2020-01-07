# 面试题

1. **ConcurrentHashMap有哪些构造函数？**

   一共有五个

   ```java
   // 无参构造函数
   public ConcurrentHashMap() {
   }
   // 可传初始容器大小的构造函数
   public ConcurrentHashMap(int initialCapacity) {
       if (initialCapacity < 0)
         throw new IllegalArgumentException();
       int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                  MAXIMUM_CAPACITY :
                  tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
       this.sizeCtl = cap;
   }
   // 可传入map的构造函数
   public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
       this.sizeCtl = DEFAULT_CAPACITY;
       putAll(m);
   }
   // 可设置初始容量、加载因子的构造函数
   public ConcurrentHashMap(int initialCapacity, float loadFactor) {
     	this(initialCapacity, loadFactor, 1);
   }
   // 可设置初始容量、加载因子、并发级别的构造函数
   public ConcurrentHashMap(int initialCapacity,
                            float loadFactor, int concurrencyLevel) {
       if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
         throw new IllegalArgumentException();
       if (initialCapacity < concurrencyLevel)   // Use at least as many bins
         initialCapacity = concurrencyLevel;   // as estimated threads
       long size = (long)(1.0 + (long)initialCapacity / loadFactor);
       int cap = (size >= (long)MAXIMUM_CAPACITY) ?
         MAXIMUM_CAPACITY : tableSizeFor((int)size);
       this.sizeCtl = cap;
   }
   ```

2. **ConcurrentHashMap使用什么技术来保证线程安全？**

   采用Node+CAS+Synchronized来保证线程安全；

4. ConcurrentHashMap 在 Java 8 中存在一个 bug 会进入死循环，原因是递归创建

5. **volatile关键字和CAS算法**

	###### 被volatile修饰的变量有如下特性:

   - 使得变量更新变得具有可见性，只要被volatile修饰的变量的赋值**一旦变化就会通知到其他线程**，如果其他线程的工作内存中存在这个同一个变量拷贝副本，那么其他线程会放弃这个副本中变量的值，重新去主内存中获取
   - 产生了内存屏障，防止指令进行了重排序
   - 总结：volatile修饰的变量具有可见性与有序性。

  ######  CAS算法
  
   - CAS的全称叫“Compare And Swap”，也就是比较与交换，他的主要操作思想是：
     首先它具有三个操作数，a、内存位置V，预期值A和新值B。如果在执行过程中，发现内存中的值V与预期值A相匹配，那么他会将V更新为新值A。如果预期值A和内存中的值V不相匹配，那么处理器就不会执行任何操作。CAS算法就是我再技术点中说的“无锁定算法”，因为线程不必再等待锁定，只要执行CAS操作就可以，会在预期中完成。

5. **锁分离技术**

6. concurrentLevel：并发级别，决定segment的个数，最大为１<<16

   换句话说，segment的个数代表了并发量





![ï¼å¾4ï¼](https://img-blog.csdnimg.cn/20181116015432609.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMyNTY4MTY=,size_16,color_FFFFFF,t_70)