# JDK1.8中HashMap源码分析详解

1. Hash算法介绍
2. HashMap源码分析
    - 构造方法
        ```java
        public HashMap() {
            // 加载因子赋值 0.75f
            this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
        }
        // initialCapacity：初始容量
        public HashMap(int initialCapacity) {
            this(initialCapacity, DEFAULT_LOAD_FACTOR);
        }
        // 1 << 30 （<<表示带符号左移：不分正负数，低位补0）
        // 0100 0000 0000 0000 0000 0000 0000 0000 = 1073741824
        // <<      :    左移运算符，num <<1,相当于num乘以2
        // >>      :    右移运算符，num >>1,相当于num除以2
        // >>>     :    无符号右移，忽略符号位，空位都以0补齐，（计算机中数字以补码存储，首位为符号位）
        //
        // 问题：为什么会是2的30次幂，而不是2的31次幂呢？
        //   答：JAVA规定了该static final类型的静态变量为int类型，由于int类型限制了该变量的长度为4个字节共32个二进制位，按理说可以向左移动31位即2的31次幂。但是事实上由于二进制数字中最高的一位也就是最左边的一位是符号位，用来表示正负之分（0为正，1为负），所以只能向左移动30位，而不能移动到处在最高位的符号位！
        static final int MAXIMUM_CAPACITY = 1 << 30;
        // initialCapacity：初始容量
        // loadFactor: 加载因子
        public HashMap(int initialCapacity, float loadFactor) {
            if (initialCapacity < 0)
                throw new IllegalArgumentException("Illegal initial capacity: " +
                                                   initialCapacity);
            // 最大容量为2的30次幂，即1073741824，
            if (initialCapacity > MAXIMUM_CAPACITY)
                initialCapacity = MAXIMUM_CAPACITY;
            // 加载因子要在0到1之间
            if (loadFactor <= 0 || Float.isNaN(loadFactor))
                throw new IllegalArgumentException("Illegal load factor: " +
                                                   loadFactor);
            this.loadFactor = loadFactor;
            // threshold是根据当前的初始化大小和加载因子算出来的边界大小，
            // 当桶中的键值对超过这个大小就进行扩容
            this.threshold = tableSizeFor(initialCapacity);
        }
        public HashMap(Map<? extends K, ? extends V> m) {
            this.loadFactor = DEFAULT_LOAD_FACTOR;
            putMapEntries(m, false);
        }
        // 求2的n次幂 >= cap (initialCapacity)
        // 根据用户传入的容量值（代码中的cap），通过计算，得到第一个比他大的2的幂并返回。
        // 其实是对一个二进制数依次向右移位，然后与原值取或。其目的对于一个数字的二进制，从第一个不为0的位开始，把后面的所有位都设置成1
        static final int tableSizeFor(int cap) {
          	// 如果传进来的数字就是2的幂自身，会出现问题，假如数字4 套用公式的话。得到的会是 8，所以才有这一行
            int n = cap - 1;
            n |= n >>> 1;
            n |= n >>> 2;
            n |= n >>> 4;
            n |= n >>> 8;
            n |= n >>> 16;
          	// 通过计算后得到的二进制后面都是1，然后加1，二进制后面都会变成0，就成了2的次幂
            return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
        }
        ```
    - 计算hashCode
        ```java
        public final int hashCode() {
            // ^ 异或运算符
            // 运算规则：两个二进制数值如果在同一位上相同，则结果中该位为0，否则为1，比如1011 & 0010 = 1001
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }
        ```
    - 计算hash
        ```java
        static final int hash(Object key) {
            int h;
            // 先计算key的hashCode值(计算出来是32位的)，然后做一次16位右位移异或混合
          	// 通过hashCode()的高16位异或低16位实现的：(h = k.hashCode()) ^ (h >>> 16)，主要是从速度、功效、质量来考虑的。以上方法得到的int的hash值，然后再通过h & (table.length -1)来得到该对象在数据中保存的位置。
            return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
        }
        ```
3. 面试题
    - **HashMap桶中链表转红黑树为什么选择数字8？**
    > 在JDK8及以后的版本中，HashMap引入了红黑树结构，其底层的数据结构变成了数组+链表或数组+红黑树。添加元素时，若桶中链表个数超过8，链表会转换成红黑树。理想情况下使用随机的哈希码，容器中节点分布在hash桶中的频率遵循泊松分布，按照泊松分布的计算公式计算出了桶中元素个数和概率的对照表，可以看到链表中元素个数为8时的概率已经非常小，再多的就更少了，所以原作者在选择链表元素个数时选择了8，是根据概率统计而选择的。

    - **HashMap的最大容量(极限容量)为什么是2的30次方？**
    > JAVA规定了该static final 类型的静态变量为int类型，至于为什么不是byte、long等类型，原因是由于考虑到HashMap的性能问题而作的折中处理！由于int类型限制了该变量的长度为4个字节共32个二进制位，按理说可以向左移动31位即2的31次幂。但是事实上由于二进制数字中最高的一位也就是最左边的一位是符号位，用来表示正负之分（0为正，1为负），所以只能向左移动30位，而不能移动到处在最高位的符号位！
    
    - **HashMap的默认长度为什么是16?**
    > HashMap会采用第一个大于该数值的2的幂作为初始化容量。
    >
    > 主要是可以使用按位与替代取模来提升hash的效率。
    >
    > 原因是这样的，如果桶初始化桶数组设置太大，就会浪费内存空间，16是一个折中的大小，既不会像1，2，3那样放几个元素就扩容，也不会像几千几万那样可以只会利用一点点空间从而造成大量的浪费。
    
    - **优化HashMap**
    > 为了最大程度的避免扩容带来的性能消耗，我们建议可以把默认容量的数字设置成expectedSize / 0.75F + 1.0F 。在日常开发中，可以使用
    > Map<String, String> map = Maps.newHashMapWithExpectedSize(10);
    
    - **HashMap默认加载因子为什么选择0.75？**
    > (折中的选择)加载因子设置为0.75而不是1，是因为设置过大，桶中键值对碰撞的几率就会越大，同一个桶位置可能会存放好几个value值，这样就会增加搜索的时间，性能下降，设置过小也不合适，如果是0.1，那么10个桶，threshold为1，你放两个键值对就要扩容，太浪费空间了。

    - **transient**
    > 一旦变量被transient修饰，变量将不再是对象持久化的一部分，该变量内容在序列化后无法获得访问。
    > transient关键字只能修饰变量，而不能修饰方法和类。注意，本地变量是不能被transient关键字修饰的。变量如果是用户自定义类变量，则该类需要实现Serializable接口。
    > 被transient关键字修饰的变量不再能被序列化，一个静态变量不管是否被transient修饰，均不能被序列化。

![ï¼å¾3ï¼](https://img-blog.csdnimg.cn/20181116015422146.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMyNTY4MTY=,size_16,color_FFFFFF,t_70)

> 注意，上图是示意图，主要是描述结构，不会达到这个状态的，因为这么多数据的时候早就扩容了。

