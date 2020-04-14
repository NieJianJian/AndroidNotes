## Java集合类

### 目录



***

***

***

### 一. 简介

Java集合类的整体框架图如下（虚线表示接口，实线表示继承）：

![图：Java集合框架图](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/javacollectionframe.jpeg)

* **Collection**包含了集合的基本操作和属性的高度抽象的接口。

  Collection包含了List和Set两大分支。

  * List是一个允许重复有序列表，主要实现类有ArrayList、LinkedList、Vector。
  * Set是一个不允许重复的无序集合，主要实现类有TreeSet、HashSet。

* **Map**是一个映射接口，即Key-Value键值对。

  * AbstractMap是个抽象类，它实现了Map接口中的大部分API。
  * Map的主要实现类有TreeMap、HashMap、LinkHashMap、WeakHashMap、ConcurrentHashMap、HashTable等。

* **Iterator**迭代器是用来遍历集合的工具。

List：有序，可重复。

* LinkedList实现了Queue接口，可作为队列使用。

Set：无序，不可重复（通过hashcode和equals保证）。

* HashSet通过HashMap实现。
* TreeSet通过TreeMap实现。
* TreeSet实现SortedSet接口，所以是有序的。

Map：key值不可重复，value值可重复。

集合总提框架主要实现类的特点表，如下：

<table>
		<tr>
  			<th>分类</th>
    		<th>实现</th>
    		<th>线程安全</th>
    		<th>排序</th>
    		<th>特点</th>
  	</tr>
  	<tr>
    		<td rowspan="4">List</td>
     	 	<td>ArrayList</td> 
     		<td>否</td>
     		<td>插入顺序</td> 
    	  <td>查找快，增删慢</td>
    </tr>
  	<tr>
      	<td>LinkedList</td> 
      	<td>否</td> 
      	<td>插入顺序</td> 
      	<td>增删快，查找慢</td>
    </tr>
  	<tr>
      	<td>Vector</td>
      	<td>是</td>
      	<td>插入顺序</td>
      	<td>并发性能不高，线程越多性能越差</td>
    </tr>
  	<tr>
      	<td>CopyOnWriteArrayList</td>
      	<td>是</td>
      	<td>插入顺序</td>
      	<td>并发性能高，占用冗余空间</td>
    </tr>
  	<tr>
      	<td rowspan="3">Set</td>
      	<td>HashSet</td>
      	<td>否</td>
      	<td>无序</td>
      	<td>同HashMap</td>
    </tr>
   	<tr>
      	<td>LinkedHashSet</td>
      	<td>否</td>
      	<td>插入顺序</td>
      	<td>同LinkedHashMap</td>
    </tr>
  	<tr>
      	<td>TreeSet</td>
      	<td>否</td>
      	<td>对象升序或降序</td>
      	<td>同TreeMap</td>
    </tr>
  	<tr>
      	<td rowspan="5">Map</td>
      	<td>HashMap</td>
      	<td>否</td>
      	<td>无序</td>
      	<td>读写性能高，接近O(1)</td>
    </tr>
  	<tr>
      	<td>LinkedHashMap</td>
      	<td>否</td>
      	<td>插入顺序</td>
      	<td>可按插入顺序遍历，性能与HashMap接近</td>
    </tr>
  	<tr>
      	<td>Hashtable</td>
      	<td>是</td>
      	<td>无序</td>
      	<td>并发性能不高，线程越多性能越差</td>
    </tr>
  	<tr>
      	<td>ConcurrentHashMap</td>
      	<td>是</td>
      	<td>无序</td>
      	<td>并发性能比HashTable高，采用分段锁</td>
    </tr>
  	<tr>
      	<td>TreeMap</td>
      	<td>否</td>
      	<td>key升序或降序</td>
      	<td>有序，读写性能O(logn)</td>
    </tr>
</table>

***

***

***

### 详细介绍

#### 1. ArrayList

* 基于数组实现，是一个动态数组，容量可以自动增长。

* 线程不安全，多线程可以考虑Collections.synchronizedList(List)函数返回一个线程安全的ArrayList。也可以使用CopyOnWriteArrayList类。

* 实现Serializable接口，支持序列化。实现RandomAccess接口，支持快速随机访问，实际上就是通过下表序号进行访问。实现Cloneable接口，可以被克隆。

* 默认初始容量10，在第一次add的时候初始化容量。

  ```java
  private void grow(int var1) {
      int var2 = this.elementData.length;
      int var3 = var2 + (var2 >> 1);
      if (var3 - var1 < 0) {
          var3 = var1;
      }
      if (var3 - 2147483639 > 0) {
          var3 = hugeCapacity(var1);
      }
      this.elementData = Arrays.copyOf(this.elementData, var3);
  }
  ```

  容量不足，每次扩容为原来的1.5倍。

  如果扩容后还不够，直接把容量变为现在需要的大小。

  最后调用copyOf将元素拷贝到新的数组中。每次add元素，都会调用grow方法，说明，每添加一个元素，就要将现有的元素拷贝一次，所以非常耗时。这就是**为什么ArrayList增删慢的原因**。如果不确定元素个数的情况，可以选择使用LinkedList。

* 查找元素可以直接通过下标获取指定的元素。在查找给定元素索引值的方法中，会判断元素是不是null，所以ArrayList是允许存放null值的。

***

#### 2. LinkedList

* 基于双向链表实现，除了可以当做链表，还可以作栈、队列、双端队列使用。

* 线程不安全。

* 实现Serializable接口，支持序列化。实现Cloneable接口，可以被克隆。

* LinkedList是基于链表实现的，因此不存在容量不足的问题，所以这里没有扩容的方法。

* 链表没有下标索引，要找到指定位置的元素，就要遍历链表

  ```java
  LinkedList.Node<E> node(int var1) {
      LinkedList.Node var2;
      int var3;
      if (var1 < this.size >> 1) {
          var2 = this.first;
          for(var3 = 0; var3 < var1; ++var3) {
              var2 = var2.next;
          }
          return var2;
      } else {
          var2 = this.last;
          for(var3 = this.size - 1; var3 > var1; --var3) {
              var2 = var2.prev;
          }
          return var2;
      }
  }
  ```

  从源码层面看，有一个加速动作。首先将index与长度size的一半进行比较，如果小于，则从0的位置开始，查找前一半，直到index。如果大于，就从最大最后往前，遍历到index处。这样可以只查找一半，一定程度上查找效率。

* 在查找和删除某元素时，源码中都划分为该元素为null和不为null两种情况来处理，LinkedList中允许元素为null。

***

#### 3. Vector

* 基于数组实现，是一个动态数组，容量可以自动增长。
* 线程安全，因为很多方法使用同步语句。
* 不支持序列化。实现了Cloneable接口，能被克隆。实现了RandomAccess接口，支持快速随机访问。
* 在构造函数中初始化容量为10。
* 无参构造函数中容量默认为10，并且设置`capacityIncrement`变量为0（该变量表示每次增加多少容量）。
* 容量不足进行扩容时，判断容量增长量参数是否为0，如果不为0，则设置新的容量为现有容量加上容量增长量。如果该参数为0，则将旧的容量翻倍。如果设置后还不够，直接把容量变为现在需要的大小。
* 很多方法添加了synchronized关键字，来保证线程安全。

***

#### 4. HashMap

* 基于哈希表实现，每一个元素是一个key-value对，内部通过单链表解决冲突问题，容量不足自动扩容。

* 线程不安全。多线程可以考虑分段锁ConcurrentHashMap。

* 实现Serializable接口，支持序列化。实现Cloneable接口，可以被克隆。

* 容量初始值为16，且实际容量必须是2的整数次幂。加载因子默认是0.75，当数据条目超过容量和加载因子的乘积，则进行resize操作，即扩容。然后对现有的条目key重新进行hash计算并存放。

  下面说下加载因子，如果加载因子越大，对空间的利用更充分，但是查找效率会降低（链表长度会越来越长）；如果加载因子太小，那么表中的数据将过于稀疏（很多空间还没用，就开始扩容了），对空间造成严重浪费。如果我们在构造方法中不指定，则系统默认加载因子为0.75，这是一个比较理想的值，一般情况下我们是无需修改的。

* HashMap的key和value都允许为null。

* **为什么容量要是2的整数次幂**？

  首先，length为2的整数次幂的话，h&(length-1)就相当于对length取模，这样便保证了散列的均匀，同时也提升了效率；其次，length为2的整数次幂的话，为偶数，这样length-1为奇数，奇数的最后一位是1，这样便保证了h&(length-1)的最后一位可能为0，也可能为1（这取决于h的值），即与后的结果可能为偶数，也可能为奇数，这样便可以保证散列的均匀性，而如果length为奇数的话，很明显length-1为偶数，它的最后一位是0，这样h&(length-1)的最后一位肯定为0，即只能为偶数，这样任何hash值都只会被散列到数组的偶数下标位置上，这便浪费了近一半的空间，因此，length取2的整数次幂，是为了使不同hash值发生碰撞的概率较小，这样就能使元素在哈希表中均匀地散列。

* Java 1.8采用红黑树结构。

***

#### 5. HashTable

* 基于哈希表实现，每一个元素是一个key-value对，内部通过单链表解决冲突问题，容量不足自动扩容。
* 线程安全。
* 实现Serializable接口，支持序列化。实现Cloneable接口，可以被克隆。
* 容量初始值为11，加载因子为0.75。扩容时容量为原来的2倍加1。
* key和value不允许为null。如果为null，编译可通过，运行时报错。

***

#### 6. LinkedHashMap

* 是HashMap的子类，存储结构一致。但它加入了一个双向链表的头结点，将所有put到LinkedHashmap的节点一一串成了一个双向循环链表，因此它保留了节点插入的顺序，可以使节点的输出顺序与输入顺序相同。
* 可以用来实现LRU算法。
* 线程不安全。
* LinkedHashMap由于继承自HashMap，因此它具有HashMap的所有特性，同样允许key和value为null。

***

#### 7. 深入HashMap

[HashMap相关问题](https://github.com/NieJianJian/AndroidNotes/blob/master/Java/HashMap原理解析.md)

[参考链接1](https://github.com/francistao/LearningNotes/blob/master/Part2/JavaSE/HashMap源码剖析.md)

