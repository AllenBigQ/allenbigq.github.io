---
layout: post
title:  "ArrayList源码分析(java1.7.0_80)"
date:   2018-1-5 23:00:00 +0000
categories: jekyll update
---


ArrayList源码分析(1.7.0_80)

### **定义**
```
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable{
            …………
        }
```

### **属性**
版本号

```
private static final long serialVersionUID = 8683452581122892189L;
```
默认的初始化容量，大小为10

```
    private static final int DEFAULT_CAPACITY = 10;
```
创建一个空的实例
```
    private static final Object[] EMPTY_ELEMENTDATA = {};

```
创建一个数组elementData用来存储ArrayList的元素

关键字：transient。在采用Java默认的序列化机制的时候，被该关键字修饰的属性不会被序列化。ArrayList采用默认序列化，因为它自己实现了序列化和反序列的方法。wreteObject(ObjectOutputStream)和readObject(ObjectInputStream)

```
    private transient Object[] elementData;
```
这个size是ArrayList中已经包含的元素个数
```
    private int size;
```

### **构造函数**

第一个构造函数

initialCapacity 最初的容量,如果传过来的起始容量小于0则报错，因为容量也不可能小于0嘛，否则就创建一个这个容量的数组赋予elementData。
```
 public ArrayList(int initialCapacity) {
        super();
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        this.elementData = new Object[initialCapacity];
    }
```

第二个构造函数
如果没有传值指定其实容量的大小，则将创建的空数组赋予elementData

```
 public ArrayList() {
        super();
        this.elementData = EMPTY_ELEMENTDATA;
    }
    
 private static final Object[] EMPTY_ELEMENTDATA = {};
```
第三个构造函数

将传递过来的集合用toArray()转换成数组赋予给elementData，计算一下此时数组大小赋予给size，如果elementData数组的类型不是Object就转换成Object类型。为什么会有这一步呢，而且源码上面有注释，是一个官方BUG，编号为6260652.

```
public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        size = elementData.length;
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    }
```
(https://bugs.java.com/bugdatabase/view_bug.do?bug_id=6260652 这个官方BUG解释说明)

也就是说c.toArray()给elementData后，elementData不一定还是Object类型了，尽管elementData定义的时候是Object类型的数组。假如我们有一个Object[]数组，并不代表着我们可以将Object对象存进去，这取决于数组中元素实际的类型，所以还要判断一下。 

Son继承Father
```
Son[] sons = new Son[]{new Son(), new Son()};
System.out.println(sons.getClass());            
            // 输出的类型是Son;
Father[] fathers = sons;
System.out.println(fathers.getClass());
            // 输出的类型是Son;
fathers[0] = new Father();                          
            // 会报错 java.lang.ArrayStoreException

http://blog.csdn.net/gulu_gulu_jp/article/details/51457492
 
```

### 主要方法

**trimToSize()**

将当前容量值设为 = 实际元素个数
```
 public void trimToSize() {
        modCount++;
        if (size < elementData.length) {
            elementData = Arrays.copyOf(elementData, size);
        }
    }
```

**ensureCapacity(int minCapacity)**

确定ArrayList的容量大小，这个函数一般用在哪里呢？一般用在ArrayList初始化的时候，能够提升效率。

```
public void ensureCapacity(int minCapacity) {
        //计算一下此时最小扩张的容量是多少
        int minExpand = (elementData != EMPTY_ELEMENTDATA)
            // any size if real element table
            ? 0
            // larger than default for empty table. It's already supposed to be
            // at default size.
            : DEFAULT_CAPACITY;
        //如果所要设置的容量大于最小扩张就需要扩大一下空间
        if (minCapacity > minExpand) {
            ensureExplicitCapacity(minCapacity);
        }
    }
```

看一下调用ensureCapacity()方法和不调用有什么区别，平时一般不会调用是因为N的数值不是特别大，通过预先设置初始化大小可以提升效率，显式的调用这个函数，对数组进行扩容。

```
public class Test1 {
    public static void main(String[] args){
        final int N = 1000000;
        Object obj = new Object();
        //没用调用ensureCapacity()方法初始化ArrayList对象
        ArrayList list = new ArrayList();
        long startTime = System.currentTimeMillis();
        for(int i=0;i<=N;i++){
            list.add(obj);
        }
        long endTime = System.currentTimeMillis();
        System.out.println("没有调用ensureCapacity()方法所用时间：" + (endTime - startTime) + "ms");

        //调用ensureCapacity()方法初始化ArrayList对象
        list = new ArrayList();
        startTime = System.currentTimeMillis();
        list.ensureCapacity(N);//预先设置list的大小
        for(int i=0;i<=N;i++){
            list.add(obj);
        }
        endTime = System.currentTimeMillis();
        System.out.println("调用ensureCapacity()方法所用时间：" + (endTime - startTime) + "ms");
    }
}
输出的结果：
没有调用ensureCapacity()方法所用时间：31ms
调用ensureCapacity()方法所用时间：14ms
```

**add(E e)**

对ArrayList集合添加一个元素

```
public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
```
添加方法会调用ensureCapacityInternal()方法，再调用ensureExplicitCapacity()方法

```
private void ensureCapacityInternal(int minCapacity) {
        if (elementData == EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }

private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        //如果最小容量大于现有的容量就需要扩容
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);      
    }
```

具体扩容的规则

```
private void grow(int minCapacity) {
        //计算出旧容量
        int oldCapacity = elementData.length;
        //计算出新容量，oldCapacity>>1 相当于除以2,所以新容量等于就容量的1.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        //新容量和实际需要的容量进行比较，如果大，则把新容量设为实际需要的容量(因为计算出的新容量太大了，不需要那么多)
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        //再和最大数组容量比较，MAX_ARRAY_SIZE=Integer.MAX_VALUE - 8
        //如果新容量大于最大容量了，就只能调用hugeCapacity()设置成最大容量，不能再大了
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```