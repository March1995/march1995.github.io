---
title: java集合框架
date: 2017-03-16 
desc:
keywords: java collection
categories: [JavaSE]
---
# Collection
![集合框架体系图](http://upload-images.jianshu.io/upload_images/4942449-cc7c288a15f6c092?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## Set (extends Collection)
Set是最简单的一种集合。集合中的对象不按特定的方式排序，并且没有重复对象。 Set接口主要实现了两个实现类：

- HashSet： HashSet类按照哈希算法来存取集合中的对象，存取速度比较快
- TreeSet ：TreeSet类实现了SortedSet接口，能够对集合中的对象进行排序。
- LinkedHashSet：具有HashSet的查询速度，且内部使用链表维护元素的顺序(插入的次序)。于是在使用迭代器遍历Set时，结果会按元素插入的次序显示。

## List (extends Collection)
实际上有两种List: 一种是基本的ArrayList,其优点在于随机访问元素，另一种是更强大的LinkedList,它并不是为快速随机访问设计的，而是具有一套更通用的方法。

List : 次序是List最重要的特点：它保证维护元素特定的顺序。List为Collection添加了许多方法，使得能够向List中间插入与移除元素(这只推荐LinkedList使用。)一个List可以生成ListIterator,使用它可以从两个方向遍历List,也可以从List中间插入和移除元素。
- ArrayList : 由数组实现的List。允许对元素进行快速随机访问，但是向List中间插入与移除元素的速度很慢。ListIterator只应该用来由后向前遍历ArrayList,而不是用来插入和移除元素。因为那比LinkedList开销要大很多。
- LinkedList : 对顺序访问进行了优化，向List中间插入与删除的开销并不大。随机访问则相对较慢。(使用ArrayList代替。)还具有下列方法：addFirst(), addLast(), getFirst(), getLast(), removeFirst() 和 removeLast(), 这些方法 (没有在任何接口或基类中定义过)使得LinkedList可以当作堆栈、队列和双向队列使用。

## Queen (extends Collection)

## Map
方法put(Object key, Object value)添加一个“值”(想要得东西)和与“值”相关联的“键”(key)(使用它来查找)。方法get(Object key)返回与给定“键”相关联的“值”。可以用containsKey()和containsValue()测试Map中是否包含某个“键”或“值”。标准的Java类库中包含了几种不同的Map：HashMap, TreeMap, LinkedHashMap, WeakHashMap, IdentityHashMap。它们都有同样的基本接口Map，但是行为、效率、排序策略、保存对象的生命周期和判定“键”等价的策略等各不相同。

执行效率是Map的一个大问题。看看get()要做哪些事，就会明白为什么在ArrayList中搜索“键”是相当慢的。而这正是HashMap提高速度的地方。HashMap使用了特殊的值，称为“散列码”(hash code)，来取代对键的缓慢搜索。“散列码”是“相对唯一”用以代表对象的int值，它是通过将该对象的某些信息进行转换而生成的。所有Java对象都能产生散列码，因为hashCode()是定义在基类Object中的方法。

HashMap就是使用对象的hashCode()进行快速查询的。此方法能够显着提高性能

Map : 维护“键值对”的关联性，使你可以通过“键”查找“值”

- HashMap : Map基于散列表的实现。插入和查询“键值对”的开销是固定的。可以通过构造器设置容量capacity和负载因子load factor，以调整容器的性能。
- LinkedHashMap : 类似于HashMap，但是迭代遍历它时，取得“键值对”的顺序是其插入次序，或者是最近最少使用(LRU)的次序。只比HashMap慢一点。而在迭代访问时发而更快，因为它使用链表维护内部次序。
- TreeMap : 基于红黑树数据结构的实现。查看“键”或“键值对”时，它们会被排序(次序由Comparabel或Comparator决定)。TreeMap的特点在于，你得到的结果是经过排序的。TreeMap是唯一的带有subMap()方法的Map，它可以返回一个子树。
- WeakHashMao : 弱键(weak key)Map，Map中使用的对象也被允许释放: 这是为解决特殊问题设计的。如果没有map之外的引用指向某个“键”，则此“键”可以被垃圾收集器回收。
- IdentifyHashMap : 使用==代替equals()对“键”作比较的hash map。专为解决特殊问题而设计

![](http://upload-images.jianshu.io/upload_images/4942449-9e051ab63d770a2f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 总结如下：
![](http://upload-images.jianshu.io/upload_images/4942449-51c43e347a13527d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 他们之间的区别：
![](http://upload-images.jianshu.io/upload_images/4942449-6c1ae30630a5a608.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 集合框架初始化容量和扩容比较：
|        | ArrayList | LinkList | HashMap | HashTable |  
| :------| :------ | :------- | :------- | :------- |
| jdk1.7 |  |  |  |   |
| 初始容量 |  |  |  |   |
| 扩容条件 |  |  |  |    |
| 扩容大小 |  |  |  |    |
| 最大容量 |  |  |  |    |
| jdk1.8 |  |  |  |   |
| 初始容量 | 10 |  | 1 << 4(16) |  11 | 
| 扩容条件 | 在第一次add时扩容检测  |  | 扩容加载因子为(0.75)，第一个临界点在当HashMap中元素的数量大于table数组长度*加载因子（16*0.75=12） | 扩容加载因子(0.75)，当超出默认长度（int）（11*0.75）=8时 |
| 扩容大小 | oldCapacity + (oldCapacity >> 1)，即大约原集合长度的1.5倍 |   | oldThr << 1(原长度*2)  | old*2+1(代码int newCapacity = (oldCapacity << 1) + 1) |
| 最大容量 | Integer.MAX_VALUE - 8 |  | 1 << 30 | Integer.MAX_VALUE - 8 |

对于java中的各种集合类，根据底层的具体实现，小结了一下大致有3种扩容的方式：

1、对于以散列表为底层数据结构实现的，（譬如hashset，hashmap，hashtable等），扩容方式为当链表数组的非空元素除以数组大小超过加载因子时，链表数组长度变大（乘以2+1），然后进行重新散列。

2、对于以数组为底层数据结构实现的，譬如ArrayList，当数组满了之后，数组长度变大 oldCapacity + (oldCapacity >> 1)，然后将原数组中的数据复制到新数组中。

3、对于以链表结构实现的，譬如TreeSet，TreeMap，则是动态增加元素~~即每次加１即可。

## 注意点：

**为啥arrayList最大值是Integer.MAX_VALUE - 8**：
数组作为一个对象，需要一定的内存存储对象头信息，对象头信息最大占用内存不可超过8字节。

## 部分源码实现过程：

### hashMap put过程

1.判断hash表是否为空，为空resize()方法创建表

2.根据hash值获得hash表中桶的头结点，若头结点为空直接调用newNode()添加结点

3.如果发生了hash冲突，先得到头结点进行比较，如果相同，替换节点值

4.如果头结点头结点不相同，且此时已处于红黑树状态，调用putTreeVal()添加入树中

5.如果还是处于链表状态，从头开始遍历链表，一旦找到相同的结点，就跳出循环，若到链表末尾，则添加一个结点

6.若添加后达到红黑树的阈值，则转换为红黑树(从treeifyBin方法可以看到，当容量小于64时，不会进行红黑树转换，只会扩容)

7.如果是添加一个数据，size将加一，如果达到阈值，则resize()扩容

### hashMap resize
  
首先获取新容量以及新阈值，然后根据新容量创建新表。如果是扩容操作，则需要进行rehash操作，通过e.hash&oldCap将链表分为两列，更好地均匀分布在新表中。 


### linkList 中间插入过程

例：在A，B两个连续结点中插入C结点,形成ACB

1.创建一个新结点，将新节点的后继指针指向B，前继指针指向A
2.将B的前指针指向C
3.根据A是否为空判断，A为空该节点为头结点，重置first结点；不为空A的后指针指向C
