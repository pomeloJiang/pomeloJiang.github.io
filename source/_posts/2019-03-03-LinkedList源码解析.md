---
title: JAVA源码-LinkedList源码分析
urlname: java_sourcecode_linkedlist
categories:
- java
tags: [JAVA]
date: 2019-03-03
---

**本文的分析基于Java 1.8源码。**

上篇分析了ArrayList的源码，点击这里:[ArrayList源码解析](./java_sourcecode_arraylist.html)
这篇将从构造方法、增删改查、遍历角度分析LinkedList源码。

LinkedList是基于链表实现的List。老规矩，先看看类图

![LinkedList](..\images\2019-03-03-LinkedList-ClassDiagram.jpg)

同ArrayList，LinkedList也是继承自AbstractList类,是Collection的子类之一，同时实现了Serializable、Cloneable接口，这就意味着：它支持序列化，能被克隆。与ArrayList不同的是，它并没有实现RandomAccess接口，也就是它不能通过下标随机访问。

与ArrayList最大的区别是，LinkedList实现了Deque接口。因此LinkedList不仅有List的特性，也有Queue的特性。

<!-- more -->

## 成员变量

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{
	private static final long serialVersionUID = 876323262645176354L;
	transient int size = 0;
    transient Node<E> first;
    transient Node<E> last;
}
```

{% raw %}
<table>
	<tr>
		<td>serialVersionUID</td>
		<td>序列化验证版本一致性字段</td>
	</tr>
	<tr>
		<td>size</td>
		<td>双向链表中节点的个数</td>
	</tr>
	<tr>
		<td>first</td>
		<td>头节点</td>
	</tr>
	<tr>
		<td>last</td>
		<td>尾节点</td>
	</tr>
</table>

{% endraw %}

上面说到，LinkedList实现了Deque的方法，其实现也是基于双向链表的特性。
成员变量first和last都是Node类型的，Node是LinkedList的内部类。定义如下:

```java
private static class Node<E> {
    //当前节点的元素
    E item;
    //下一个节点
    Node<E> next;
    //上一个节点
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

</br>

## 构造函数

```java
public LinkedList() {}
public LinkedList(Collection<? extends E> c) {
    this();
    addAll(c);
}
```

LinkedList有两个构造

1. LinkedList():空构造

2. LinkedList(Collection<? extends E> c):将传入集合c中的元素添加到链表尾部。

我们来看看非空构造的addAll()方法
addAll()最终调用的是带两个参数的重载。传入参数size，将指定集合里的元素，依次添加到链表的尾部。而addAll(int index, Collection<? extends E> c) 是将指定集合里的元素，依次添加到index位置以后的链表节点。

```java
public boolean addAll(Collection<? extends E> c) {
    return addAll(size, c);
}
public boolean addAll(int index, Collection<? extends E> c) {
    //下标越界检查
    checkPositionIndex(index);
    //将传入的集合转为数组类型
    Object[] a = c.toArray();
    int numNew = a.length;
    if (numNew == 0)
        return false;

    //succ为index当前的节点，pred是当前节点的上一个节点
    Node<E> pred, succ;
   	//如果从尾部插入
    if (index == size) {
        succ = null;
        pred = last;
    } else {
        //从链表头部或中间插入
        succ = node(index);
        pred = succ.prev;
    }

    //将集合a中的元素依次插入链表
    for (Object o : a) {
        @SuppressWarnings("unchecked") E e = (E) o;
        Node<E> newNode = new Node<>(pred, e, null);
        //pred == null是从头部插入
        if (pred == null)
            first = newNode;
        else
            //将新节点设为pred的下一个节点
            pred.next = newNode;
        pred = newNode;
    }

    //从尾部插入的话，需要处理last指针
    if (succ == null) {
        last = pred;
    } else {
        pred.next = succ;
        succ.prev = pred;
    }

    size += numNew;
    modCount++;
    return true;
} 
```

addAll主要做了以下几件事:
1.检查下标越界
2.将集合转为数组类型
3.在链表的指定位置依次插入集合中的元素
4.改变size大小

</br>

## 常用操作

LinkedList继承自AbstractList，又实现了Deque。因此同时有List和Queue的方法。这里将分别分析属于List和Queue的操作并给出时间复杂度。

LinkedList有几个linkXxx和unlinkXxx的基础操作，后面实现的属于List的和Queue的几个方法都是基于这几个方法来实现的。

### LinkedList的基础方法
我们先分别看看这几个基础方法。除了node()方法外，其他方法的时间复杂度都为O(1)

#### node()

查找指定位置的节点

```
Node<E> node(int index) {
	//如果 index < size/2 则从0开始查找指定下标的节点
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
    	//如果index >= size/2 则从size-1向前查找指定下标的节点
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

#### linkFirst()

在头部插入节点

```java
private void linkFirst(E e) {
    final Node<E> f = first;
    //头结点作为新建节点的后一个节点
    final Node<E> newNode = new Node<>(null, e, f);
    //当前节点指向first指针
    first = newNode;
    if (f == null)
        last = newNode;
    else
        //原来的头节点的上一个节点指向新建节点
        f.prev = newNode;
    size++;
    modCount++;
}
```

#### linkLast()

在尾部插入节点

```java
void linkLast(E e) {
    final Node<E> l = last;
    //尾节点作为新建节点的上一个节点
    final Node<E> newNode = new Node<>(l, e, null);
    //当前节点指向last指针
    last = newNode;
    if (l == null)
        first = newNode;
    else
        //原来的尾节点的下一个节点指向新节点
        l.next = newNode;
    size++;
    modCount++;
}
```

#### linkBefore()

在指定位置前插入节点

```java
void linkBefore(E e, Node<E> succ) {
    //假设succ不为null，保存succ节点的上一个节点为pred
    final Node<E> pred = succ.prev;
    //新建节点，将pred和succ分别作为新节点的上一个和下一个节点
    final Node<E> newNode = new Node<>(pred, e, succ);
    succ.prev = newNode;
    if (pred == null)
        first = newNode;
    else
        pred.next = newNode;
    size++;
    modCount++;
}
```

#### unlinkFirst()

删除头节点

```java
private E unlinkFirst(Node<E> f) {
    //假设f为头节点且f不为null
    final E element = f.item;
    final Node<E> next = f.next;
    f.item = null;
    f.next = null; // help GC
    //将first指针指向f的下一个节点
    first = next;
    //处理链表只有一个头节点的情况
    if (next == null)
        last = null;
    else
        next.prev = null;
    size--;
    modCount++;
    return element;
}
```

#### unlinkLast()

删除尾节点

```java
private E unlinkLast(Node<E> l) {
    //假设l为尾节点且尾节点不为null
    final E element = l.item;
    final Node<E> prev = l.prev;
    l.item = null;
    l.prev = null; // help GC
    //将last指针指向尾节点的上一个节点
    last = prev;
    //处理链表只有一个尾节点的情况
    if (prev == null)
        first = null;
    else
        prev.next = null;
    size--;
    modCount++;
    return element;
}
```

#### unlink()

删除指定的节点

```java
 E unlink(Node<E> x) {
     //假设传入的节点不为null
     final E element = x.item;
     //保存当前节点的下一个节点，上一个节点
     final Node<E> next = x.next
     final Node<E> prev = x.prev
     //如果传入的节点没有上一个节点，则传入的节点为头节点
     if (prev == null) {
         first = next;
     } else {
         //将prev的next指针指向当前节点的next节点
         prev.next = next;
         x.prev = null;
     }
     //如果传入的节点没有下一个节点，则传入的节点为尾节点
     if (next == null) {
         last = prev;
     } else {
         //将next的prev指针指向当前节点的prev节点
         next.prev = prev;
         x.next = null;
     }
     x.item = null;
     size--;
     modCount++;
     return element;
 }
```

### List部分的方法

#### add()

添加元素

```java
public boolean add(E e) {
    linkLast(e);
    return true;
}
//从指定位置插入元素
public void add(int index, E element) {
    //下标越界检查
    checkPositionIndex(index);
    if (index == size)
        linkLast(element);
    else
        linkBefore(element, node(index));
}
```

1.add(E e):通过linkLast(E e)在链表尾节点添加一个新节点，并将last指针指向新节点。 
2.add(int index, E element):1、下标越界检查  2、判断指定位置是否是尾部,成立调用linkLast()在尾部插入新节点，反之调用linkBefore()在指定位置插入新节点。
3.时间复杂度:O(1)

#### remove()

移除元素

```java
public E remove(int index) {
    //下标越界检查
    checkElementIndex(index);
    return unlink(node(index));
}
public boolean remove(Object o) {
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}
```

1.remove(int index): 1、下标越界检查  2、调用unlink()在指定位置删除节点并返回删除的节点
2.remove(Object o):  1、判断传入的对象是否为null 2、调用unlink移除指定节点
3.时间复杂度O(n)

#### size()

计算大小，直接返回size变量

```java
public int size() {
	return size;
}
```



### Queue部分的方法

#### peek()

返回头节点但不移除，头节点为null就返回null，时间复杂度O(1)

```java
public E peek() {
    final Node<E> f = first;
    return (f == null) ? null : f.item;
}
```

#### element()

获取头节点但不移除，若头节点为null的话就抛出异常，时间复杂度O(1)

```java
public E element() {
    return getFirst();
}
public E getFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return f.item;
}
```

#### poll()

移除头节点，头节点为null就返回null，时间复杂度O(1)

```java
public E poll() {
    final Node<E> f = first;
    return (f == null) ? null : unlinkFirst(f);
}
```

#### remove()

移除头节点，若头节点为null的话就抛出异常，时间复杂度O(1)

```java
public E remove() {
    return removeFirst();
}
```

#### offer()

在尾部添加新节点，时间复杂度O(1)

```java
public boolean offer(E e) {
    return add(e);
}
```

#### push()

```java
public void push(E e) {
    addFirst(e);
}
```

#### pop()

```java
public E pop() {
    return removeFirst();
}
```



## 遍历

## 总结

