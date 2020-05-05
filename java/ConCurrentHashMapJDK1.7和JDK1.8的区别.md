**ConCurrentHashMap 1.8 相比 1.7的话，主要改变为：**

- 去除 `Segment + HashEntry + Unsafe` 的实现，
  改为 `Synchronized + CAS + Node + Unsafe` 的实现
  其实 Node 和 HashEntry 的内容一样，但是HashEntry是一个内部类。
  用 Synchronized + CAS 代替 Segment ，这样锁的粒度更小了，并且不是每次都要加锁了，CAS尝试失败了在加锁。
- put()方法中 初始化数组大小时，1.8不用加锁，因为用了个 `sizeCtl` 变量，将这个变量置为-1，就表明table正在初始化。

**下面简单介绍下主要的几个方法的一些区别：**

## 1. put() 方法

**JDK1.7中的实现：**

ConCurrentHashMap 和 HashMap 的put()方法实现基本类似，所以主要讲一下为了实现并发性，ConCurrentHashMap 1.7 有了什么改变

- 需要定位 2 次 （segments[i]，segment中的table[i]）
  由于引入segment的概念，所以需要

1. 先通过key的 `rehash值的高位` 和 `segments数组大小-1` 相与得到在 segments中的位置
2. 然后在通过 `key的rehash值` 和 `table数组大小-1` 相与得到在table中的位置

- 没获取到 segment锁的线程，没有权力进行put操作，不是像HashTable一样去挂起等待，而是会去做一下put操作前的准备：

1. table[i]的位置(你的值要put到哪个桶中)
2. 通过首节点first遍历链表找有没有相同key
3. 在进行1、2的期间还不断自旋获取锁，超过 `64次` 线程挂起！

**JDK1.8中的实现：**

- 先拿到根据 `rehash值` 定位，拿到table[i]的 `首节点first`，然后：

1. 如果为 `null` ，通过 `CAS` 的方式把 value put进去
2. 如果 `非null` ，并且 `first.hash == -1` ，说明其他线程在扩容，参与一起扩容
3. 如果 `非null` ，并且 `first.hash != -1` ，Synchronized锁住 first节点，判断是链表还是红黑树，遍历插入。

## 2. get() 方法

**JDK1.7中的实现：**

- get操作实现非常简单和高效。先经过一次再哈希，然后使用这个哈希值通过哈希运算定位到segment，再通过哈希算法定位到元素
- get方法里将要使用的共享变量都定义成volatile，如用于统计当前Segement大小的count字段和用于存储值的HashEntry的value，所以并不用加锁，valatile保证了共享变量的可见性，所以支持多线程读，但是只支持单线程写(有一种情况可以被多线程写，就是写入的值不依赖于原值)
- 由于变量 `value` 是由 `volatile` 修饰的，java内存模型中的 `happen before` 规则保证了 对于 volatile 修饰的变量始终是 `写操作` 先于 `读操作` 的，并且还有 volatile 的 `内存可见性` 保证修改完的数据可以马上更新到主存中，所以能保证在并发情况下，读出来的数据是最新的数据。

**JDK1.8中的实现：**

- 1.计算hash值，定位到该table索引位置，如果是首节点符合就返回
- 2.如果遇到扩容的时候，会调用标志正在扩容节点ForwardingNode的find方法，查找该节点， 匹配就返回
- 3.以上都不符合的话，就往下遍历节点，匹配就返回，否则最后就返回null
- 和JDK1.7一样，value是用volatile修饰的，所以get操作不用加锁。

## 3. resize() 方法

**JDK1.7中的实现：**

- 跟HashMap的 resize() 没太大区别，都是在 put() 元素时去做的扩容，所以在1.7中的实现是获得了锁之后，在单线程中去做扩容（1.`new个2倍数组` 2.`遍历old数组节点搬去新数组`）。

**JDK1.8中的实现：**

- jdk1.8的扩容支持并发迁移节点，从old数组的尾部开始，如果该桶被其他线程处理过了，就创建一个 ForwardingNode 放到该桶的首节点，hash值为-1，其他线程判断hash值为-1后就知道该桶被处理过了。

## 4. 计算size

**JDK1.7中的实现：**

1. 先采用不加锁的方式，计算两次，如果两次结果一样，说明是正确的，返回。
2. 如果两次结果不一样，则把所有 segment 锁住，重新计算所有 segment的 `Count` 的和

**JDK1.8中的实现：**

由于没有segment的概念，所以只需要用一个 `baseCount` 变量来记录ConcurrentHashMap 当前 `节点的个数`。

1. 先尝试通过CAS 修改 `baseCount`
2. 如果多线程竞争激烈，某些线程CAS失败，那就CAS尝试将 `CELLSBUSY` 置1，成功则可以把 `baseCount变化的次数` 暂存到一个数组 `counterCells` 里，后续数组 `counterCells` 的值会加到 `baseCount` 中。
3. 如果 `CELLSBUSY` 置1失败又会反复进行CAS `baseCount` 和 CAS `counterCells`数组