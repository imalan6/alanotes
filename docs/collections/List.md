#### ArrayList

![ArrayList-1-768x406-1.png](https://i.loli.net/2021/02/23/58BcsDv2uzxdkOp.png)

ArrayList底层使用的是Object数组存储元素。

添加元素

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
}
```

ensureCapacityInternal方法主要用于检查当前List的容量是否已经达到上限，如果达到上限需要进行扩容。然后是添加元素的逻辑，直接将新元素到添加到elementData数组中即可。

###### 添加指定元素到指定位置

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    public void add(int index, E element) {
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }
}
```

首先校验给定的添加位置index，index必须小于size的值，并大于等于0。然后同样校验容量是否达到上限，达到上限需要扩容。

之后执行一个本地方法，System.arraycopy方法用于将指定位置及其后面的所有元素整个后移一位，复制迁移到从index+1开始的位置，将index位空出来用于插入新元素，最后将新元素添加到空出的index位置。

###### 扩容机制

众所周知，数组一旦被初始化，其长度就固定了，不可以再改变。但是ArrayList却可以动态的添加元素。这就产生了ArrayList中的一个重要特性——扩容。

实现机制：ArrayList.ensureCapacity(int minCapacity)

add和addAll方法中多次出现的ensureCapacityInternal方法就是通向扩容逻辑的通道。

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    // 确保底层数组的容量足够保存当前的元素或元素集，如果容量不足进行扩容。
    private void ensureCapacityInternal(int minCapacity) {
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
    }
    // 处理首次添加元素时的容量扩容操作，被指定为DEFAULTCAPACITY_EMPTY_ELEMENTDATA的空数组在首次添加元素时需要自动扩容到默认容量10
    private static int calculateCapacity(Object[] elementData, int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        return minCapacity;
    }
    // 校验是否需要扩容，只有当给定容量值比当前数组的长度要大时，才需要扩容，
    // 因为一般情况下给定容量即为新添加元素后的容量，当前容量达不到这个值是没有位置保存当前元素的，所以才需要扩容。
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
}
```

ensureCapacityInternal方法的目的是确保给定的参数指定的容量值。

扩容逻辑在grow方法中：

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);// 扩容为原容量的1.5倍
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        // 如果最后决定扩容的容量比允许的最大数组容量值要大，那么则进行超限处理
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
    // 处理超限问题
    // 如果给定的minCapacity为负数（首位为1）则抛出异常错误OutOfMemoryError
    // 如果给定容量大于数组最大容量，则取整数的最大值为容量，否则使用数组的最大容量作为扩容容量
    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
}
```

首先要根据规则计算一个新容量newCapacity（(oldCapacity * 3)/2 + 1），然后将这个新容量值与给定需要的容量值minCapacity进行比较，如果新容量值大于给定容量值，则用新容量值进行扩容，否则使用给定容量值进行扩容。然后进行超限校验和处理。

最后使用确定好的容量newCapacity来作为新的底层数组容量来进行扩容操作：创建一个新的数组，并迁移元素。
ArrayList底层使用的是Object数组，所以ArrayList具有数组查询速度快的优点以及增删速度慢的缺点，且线程不安全。



#### LinkedList

![302258007343926_0_.jpg](https://i.loli.net/2021/02/23/uWQR7ibej83yvUh.jpg)

在LinkedList的底层是一种双向循环链表。在此链表上每一个数据节点都由三部分组成：前指针（指向前面的节点的位置），数据，后指针（指向后面的节点的位置）。最后一个节点的后指针指向第一个节点的前指针，形成一个循环。

数据结构原理

LinkedList底层的数据结构是基于双向循环链表的，且头结点中不存放数据,如下：

![050153010163922.jpg](https://i.loli.net/2021/02/23/2z7ntA6BDouYhlX.jpg)

 

既然是双向链表，那么必定存在一种数据结构——我们可以称之为节点，节点实例保存业务数据，前一个节点的位置信息和后一个节点位置信息，如下图所示：

![050101321255262.png](https://i.loli.net/2021/02/23/1louWZRYiAxHF8t.png)

私有属性

LinkedList中定义了两个属性：

```java
1 private transient Entry<E> header = new Entry<E>(null, null, null);
2 private transient int size = 0;
```

header是双向链表的头节点，它是双向链表节点所对应的类Entry的实例。Entry中包含成员变量： previous, next, element。其中，previous是该节点的上一个节点，next是该节点的下一个节点，element是该节点所包含的值，size是双向链表中节点实例的个数。entry类定义如下：

```java
private static class Entry<E> {
   E element;
    Entry<E> next;
    Entry<E> previous;

    Entry(E element, Entry<E> next, Entry<E> previous) {
        this.element = element;
        this.next = next;
        this.previous = previous;
   }
}
```

节点类很简单，element存放业务数据，previous与next分别存放前后节点的指针。

添加元素

```java
//普通的在尾部添加元素
public boolean add(E e) {
    linkLast(e);
    return true;
}

//在指定位置添加元素
public void add(int index, E element) {
    checkPositionIndex(index);
    //指定位置也有可能是在尾部
    if (index == size)
        linkLast(element);
    else
        linkBefore(element, node(index));
}

public boolean addAll(Collection<? extends E> c) {
    return addAll(size, c);
}
// index参数指定collection中插入的第一个元素的位置
public boolean addAll(int index, Collection<? extends E> c) {
    // 插入位置超过了链表的长度或小于0，报IndexOutOfBoundsException异常
    if (index < 0 || index > size)
        throw new IndexOutOfBoundsException("Index: "+index+
                                                ", Size: "+size);
    Object[] a = c.toArray();
   int numNew = a.length;
   // 若需要插入的节点个数为0则返回false，表示没有插入元素
    if (numNew==0)
        return false;
    modCount++;//否则，插入对象，链表修改次数加1
    // 保存index处的节点。插入位置如果是size，则在头结点前面插入，否则在获取index处的节点插入
    Entry<E> successor = (index==size ? header : entry(index));
    // 获取前一个节点，插入时需要修改这个节点的next引用
    Entry<E> predecessor = successor.previous;
    // 按顺序将a数组中的第一个元素插入到index处，将之后的元素插在这个元素后面
    for (int i=0; i<numNew; i++) {
        // 结合Entry的构造方法，这条语句是插入操作，相当于C语言中链表中插入节点并修改指针
        Entry<E> e = new Entry<E>((E)a[i], successor, predecessor);
        // 插入节点后将前一节点的next指向当前节点，相当于修改前一节点的next指针
        predecessor.next = e;
        // 相当于C语言中成功插入元素后将指针向后移动一个位置以实现循环的功能
        predecessor = e;
  }
    // 插入元素前index处的元素链接到插入的Collection的最后一个节点
    successor.previous = predecessor;
    // 修改size
    size += numNew;
    return true;
}
```

删除元素

```java
private E remove(Entry<E> e) {
    if (e == header)
        throw new NoSuchElementException();
    // 保留将被移除的节点e的内容
    E result = e.element;
   // 将前一节点的next引用赋值为e的下一节点
    e.previous.next = e.next;
   // 将e的下一节点的previous赋值为e的上一节点
    e.next.previous = e.previous;
   // 上面两条语句的执行已经导致了无法在链表中访问到e节点，而下面解除了e节点对前后节点的引用
   e.next = e.previous = null;
  // 将被移除的节点的内容设为null
  e.element = null;
  // 修改size大小
  size--;
  modCount++;
  // 返回移除节点e的内容
  return result;
}
```

双向循环链表的查询效率低但是增删效率高。

