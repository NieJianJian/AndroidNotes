

必问

* 你用过hashmap吗？什么是hashmap？你为什么用它？

* 你知道hashmap的工作原理吗？

* 你知道hashmap的get()方法的工作原理吗？

* 当两个对象的hashcode相同会发生什么？

* **如果两个键的hashcode相同，你如何获取值对象？**

  节点包含了key和hashcode字段，即使hashcode相同，key只要不一致，就可以获取值对象。

* **如果hashmap的大小超过负载因子（load factor）定义的容量，怎么办？**

  table 大小 16、32。

  ```
  static final float DEFAULT_LOAD_FACTOR = .75F; 
  ```

  16 * DEFAULT_LOAD_FACTOR = 12;  // 阈值，优化

  时空转换，如果factor越小，说明碰撞的可能性越低，浪费的内存越多，factor越大，碰撞的可能性越大，浪费的内存越小。0.6 ~ 0.75 比较合理。

* **你了解重新调整HashMap大小存在什么问题吗？**

  说明更多的空间，将会被浪费。doubleCapacity()方法，将oldCapacity * 2，之前存储的值，都需要重新hash，进行调整。

* **为什么String、Integer这样的Wrapper类适合做为键？**

  Int,float是基本数据类型，基本数据类型不是对象，没有继承Object，所以没有hashCode()

* **我们可以使用自定义的对象做为键吗？**

  可以，重写hashcode()。String.class中有hashcode()重写方法，可以参考。

* **hashmap的初始数组大小是如何决定的？**

* **hashmap初始大小是否可以自定义？**

  ```java
  public HashMap() {
  		table = (HashMapEntry<K, V>[]) EMPTY_TABLE;
  		threshold = -1; // Forces first put invocation to replace EMPTY_TABLE
  }
  private static final Entry[] EMPTY_TABLE = new HashMapEntry[MINIMUM_CAPACITY >>> 1];
  private static final int MINIMUM_CAPACITY = 4;
  ```

  hashmap数组初始大小为2。

* **hashmap数组大小有没有上限？**

  ```java
  private static final int MAXIMUM_CAPACITY = 1 << 30; // 最大容量上限
  ```

* **hashmap何时扩容，如何扩容？**

  ```java
  if (size++ > threshold) {
      tab = doubleCapacity();
      index = hash & (tab.length - 1);
  }
  ```

  当存储的个数，超过阈值，执行扩容，直接翻倍int newCapacity = oldCapacity * 2;

  阈值threshold = capacity * load factor；

* **hashmap链表插入在表头还是表尾？**

  ```java
  void addNewEntry(K key, V value, int hash, int index) {
      table[index] = new HashMapEntry<K, V>(key, value, hash, table[index]);
  }
  ```

  有上面的代码可见，插入到表头，因为HashMapEntry的构造函数最后一个参数为next。

* **HashMap的容量为什么必须是2的幂次方**

  2的幂次方的二进制是1111结尾，可以减少碰撞的概率，充分利用资源。

* **HashMap的key是否可以为null，null如何存储。**

  key可以为null，







* 数组和链表如何组织工作？

* int hash是什么？有什么作用？

* Hash的原理是什么？

* **Hash的put方法原理？**

  1.根据key计算hash值

  2.hash值进行位运算，得到index

  3.for循环遍历，根据index，获取链表节点，然后判断链表节点的hash值，和此时需要put操作的key计算出的hash值，是否一致，然后在判断该节点 的key，是否等于put操作的key。两者都满足，说明此时，key已经存在于链表中，就重新复制该key对应的value值。

  4.如果第三步遍历没有重复的key，则添加新的节点。

* **Hash的get方法原理？**

  1.通过key计算hash值

  2.通过hash值进行位运算，得到index

  3.进行轮询，查找key和hash都满足的节点，取出value值

* **hashCode为什么计算复杂？**

* **什么是hash碰撞**

* **为什么节点要包含key、value、hashcode、next？**

  节点为什么要包含key和hashcode，是用于put操作时，进行校验key是否已经存在了。

  ```java
  if (e.hash == hash && key.equals(e.key)) {
  ```

 

* android sdk 28，将链表转换为红黑树。

  在jdk1.8版本后，java对HashMap做了改进，在链表长度大于8的时候，将后面的数据存在红黑树中，以加快检索速度，



Int index = hash & (tab.length - 1)    ==   hash % (tab.length -1)

key通过hash运算得到int类型的hash值，然后hash值再通过&运算，得到tab数组大小区间内的下标值。

hash碰撞：通过hash值计算的index相同，称为hash碰撞。



ThreadLocal / SparseArray



* 图片上传怎么做？

  不能说：接口怎么实现，我就怎么调用，虽然我们都是这么做的。

  明白什么是http，从而知道http如何上传图片，三次握手，四次挥手，



#### 参考链接

1.[Java8系列之重新认识HashMap](https://zhuanlan.zhihu.com/p/21673805)





