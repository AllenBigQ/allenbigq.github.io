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

关键字：transient。在采用Java默认的序列化机制的时候，被该关键字修饰的属性不会被序列化。ArrayList不采用默认序列化，因为它自己实现了序列化和反序列的方法。wreteObject(ObjectOutputStream)和readObject(ObjectInputStream)

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
**add(int index, E element)**
```
public void add(int index, E element) {
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }
```


**size()**

返回ArrayList实际大小
```
public int size() {
        return size;
    }
```

**isEmpty()**
判断当前ArrayList是否为空
```
public boolean isEmpty() {
        return size == 0;
    }
```
**contains(Object o)和indexOf(Object o)**

contains(Object o) 判断ArrayList是否包含Object o
indexOf(Object o) 正向查找，返回元素的索引值，没有则返回-1
```
public boolean contains(Object o) {
        return indexOf(o) >= 0;
    }
    
public int indexOf(Object o) {
        if (o == null) {
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = 0; i < size; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }
```
**lastIndexOf(Object o)**

反向查找(从数组末尾向开始查找)，返回元素的索引值
```
public int lastIndexOf(Object o) {
        if (o == null) {
            for (int i = size-1; i >= 0; i--)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = size-1; i >= 0; i--)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }
```
**clone()**

克隆函数
```
public Object clone() {
        try {
            @SuppressWarnings("unchecked")
                ArrayList<E> v = (ArrayList<E>) super.clone();
            // 将当前ArrayList的全部元素拷贝到v中
            v.elementData = Arrays.copyOf(elementData, size);
            v.modCount = 0;
            return v;
        } catch (CloneNotSupportedException e) {
            // this shouldn't happen, since we are Cloneable
            throw new InternalError();
        }
    }
```
**toArray()**

返回ArrayList的Object数组

```
public Object[] toArray() {
        return Arrays.copyOf(elementData, size);
    }
```

**toArray(T[] a)**

调用 toArray() 函数会抛出“java.lang.ClassCastException”异常，但是调用 toArray(T[] contents) 能正常返回 T[]。

toArray() 会抛出异常是因为 toArray() 返回的是 Object[] 数组，将 Object[] 转换为其它类型(如如，将Object[]转换为的Integer[])则会抛出“java.lang.ClassCastException”异常，因为Java不支持向下转型。
解决该问题的办法是调用 <T> T[] toArray(T[] contents) ， 而不是 Object[] toArray()。


```
public <T> T[] toArray(T[] a) {
        // 若数组a的大小 < ArrayList的元素个数；
        // 则新建一个T[]数组，数组大小是“ArrayList的元素个数”，并将“ArrayList”全部拷贝到新数组中
        if (a.length < size)
            return (T[]) Arrays.copyOf(elementData, size, a.getClass());
        // 若数组a的大小 >= ArrayList的元素个数；
        // 则将ArrayList的全部元素都拷贝到数组a中。
        System.arraycopy(elementData, 0, a, 0, size);
        if (a.length > size)
            a[size] = null;
        return a;
    }
//使用方法：
public static Integer[] vectorToArray2(ArrayList<Integer> v) {
    Integer[] newText = (Integer[])v.toArray(new Integer[0]);
    return newText;
}
```

**get(int index)和elementData(int index)**

获取index位置的元素
```
public E get(int index) {
        rangeCheck(index);

        return elementData(index);
    }

E elementData(int index) {
        return (E) elementData[index];
    }

private void rangeCheck(int index) {
    //如果index大于ArrayList的实际大小则抛出异常    
    if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }
```
**set(int index, E element)**

设置index位置的值为element

```
public E set(int index, E element) {
        rangeCheck(index);

        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }
```
**remove(int index)和remove(Object o)**

移除index索引的元素和直接移除元素

```
public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }

    
public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            // 遍历ArrayList，找到“元素o”，则删除，并返回true。
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
}
//快速删除
private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }
```
**clear()**

清空ArrayList

```
public void clear() {
        modCount++;

        // clear to let GC do its work
        for (int i = 0; i < size; i++)
            elementData[i] = null;

        size = 0;
    }
```
**addAll(Collection<? extends E> c)和addAll(int index, Collection<? extends E> c)**

将集合c追加到ArrayList中/从index位置开始，将集合c添加到ArrayList
```
public boolean addAll(Collection<? extends E> c) {
        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  // Increments modCount
        System.arraycopy(a, 0, elementData, size, numNew);
        size += numNew;
        return numNew != 0;
    }

public boolean addAll(int index, Collection<? extends E> c) {
        rangeCheckForAdd(index);

        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  // Increments modCount

        int numMoved = size - index;
        if (numMoved > 0)
            System.arraycopy(elementData, index, elementData, index + numNew,
                             numMoved);

        System.arraycopy(a, 0, elementData, index, numNew);
        size += numNew;
        return numNew != 0;
    }
    
private void rangeCheckForAdd(int index) {
        if (index > size || index < 0)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }
```
**removeRange(int fromIndex, int toIndex)**

删除fromIndex到toIndex之间的全部元素。

```
protected void removeRange(int fromIndex, int toIndex) {
        modCount++;
        int numMoved = size - toIndex;
        System.arraycopy(elementData, toIndex, elementData, fromIndex,
                         numMoved);

        // clear to let GC do its work
        int newSize = size - (toIndex-fromIndex);
        for (int i = newSize; i < size; i++) {
            elementData[i] = null;
        }
        size = newSize;
    }
```

**removeAll(Collection<?> c)和retainAll(Collection<?> c)**
删除在指定集合中的元素/删除不在指定集合的元素
```
public boolean removeAll(Collection<?> c) {
        return batchRemove(c, false);
    }
public boolean retainAll(Collection<?> c) {
        return batchRemove(c, true);
    }
private boolean batchRemove(Collection<?> c, boolean complement) {
        final Object[] elementData = this.elementData;
        int r = 0, w = 0;
        boolean modified = false;
        try {
            for (; r < size; r++)
                if (c.contains(elementData[r]) == complement)
                    elementData[w++] = elementData[r];
        } finally {
            // Preserve behavioral compatibility with AbstractCollection,
            // even if c.contains() throws.
            if (r != size) {
                System.arraycopy(elementData, r,
                                 elementData, w,
                                 size - r);
                w += size - r;
            }
            if (w != size) {
                // clear to let GC do its work
                for (int i = w; i < size; i++)
                    elementData[i] = null;
                modCount += size - w;
                size = w;
                modified = true;
            }
        }
        return modified;
    }
```

### **ArrayList的遍历方式**

**第一种，通过迭代器遍历。即通过Iterator去遍历**
```
Integer value = null;
Iterator iter = list.iterator();
while (iter.hasNext()) {
    value = (Integer)iter.next();
}
```
**第二种，随机访问，通过索引值去遍历**

由于ArrayList实现了RandomAccess接口，它支持通过索引值去随机访问元素。
```
Integer value = null;
int size = list.size();
for (int i=0; i<size; i++) {
    value = (Integer)list.get(i);        
}
```
**第三种，for循环遍历**
```
Integer value = null;
for (Integer integ:list) {
    value = integ;
}
```

**总结：遍历ArrayList时，使用随机访问(即，通过索引序号访问)效率最高，而使用迭代器的效率最低！**