# ConcurrentSkipListSet

**ConcurrentSkipListSet**的实现非常简单，其内部引用了一个`ConcurrentSkipListMap`对象，所有API方法均委托`ConcurrentSkipListMap`对象完成