---
title: Java-ArrayList-源码分析（一）
date: 2020-06-07
categories: Java
tags: [Java, ArrayList, 源码分析]
---

ArrayList是数组实现的顺序表，是泛型Collection Framwork中的一员。本博文尝试分析一下ArrayList的源码，主要是弄懂实现背后的原理，搞清楚为什么要那么实现。为此，采用循序渐进，逐步深入的分析方式。

- 先自己简单实现一遍
- 对照库的实现，挨个分析库为什么要那么写，那么写有什么好处。
- 库实现，涉及到哪些巧妙的方法。

仔细分析起来，内容还是蛮多。为了不繁杂，博文可能需要分成多个部分，每个部分分析ArrayList中一簇功能。
本篇第一篇，先总体概览，再说说ArrayList Constructor。

<!--more-->

# 顺序表怎么实现

先考虑一下，如果不去看ArrayList的库实现，要自己写一个ArrayList，我们要怎么做。

## 定义要实现的ArrayList功能。

既然是个顺序表，所需功能大体离不开增删改查，再加上构造函数。

### 构造

- 构造一个空的表。
- 构造一个有一定容量的表。
- 通过一个其它集合构造一个顺序表，将集合中元素挨个塞进顺序表。

### 增

顺序表增加的操作，一般来说，有：

- 在末尾增加一个元素。
- 在某个位置增加一个元素。
- 将其它集合里面的元素批量增加到末尾。
- 将其它集合里面的元素批量增加到某个位置。

### 删

顺序表删除的操作，一般来说，有：

- 删除某个位置上的元素。
- 删除一个位置区间内所有元素。
- 从顺序表中删除值为x的个元素。
- 删除所有元素， 即清空。

### 改

顺序表修改的操作，有：

- 修改某个位置上的元素。

### 查

- 获取某个位置上的元素。
- 查找某个元素在顺序表中的位置。
- 判断值为x的元素是否包含在顺序表中。

## 初步实现

由于类功能较简单，我们就直接写代码原型。

```
package mypackage;

public class MyArrayList<E> {
    private E[] elementData;
    private int size;
    
    // 创建空的list
    public MyArrayList() {
        elementData = new E[0];
        size = 0;
    }
    
    // 创建一个容量为capacity的list，为什么需要这样一个初始容量的构造函数？
    // 只提供默认空构造函数，然后调用增加元素方法进行内部数组申请，不也行么？
    public MyArrayList(int capacity) {
        elementData = new E[capacity];
        size = 0;
    }
    
    // 基于其它集合创建顺序表
    public MyArrayList(MyArrayList<E> c) {
        System.arraycopy(c.elementData, 0, elementData, 0, c.size());
        size = c.size();
    }
    
    // 在末尾增加元素
    public void add(E e) {
        if (elementData.length == size) {
            ensureCapacity(size + 1);
        }
        elementData[size++] = e;
    }
    
    // 在某个位置增加元素，要判断参数合法性，如果不合法，是抛异常呢，还是返回错误值呢
    public void add(int index, E e) {
        // check arguments,校验index是否 < 0 或者 > size
        // ...
        
        if (elementData.length == size) {
            ensureCapacity(size + 1);
        }
        
        // 将index及其后面的元素都往后错一位。System.arraycopy支持区间重叠拷贝。
        System.arraycopy(...);
        elementData[index] = e;
        size += 1;
    }
    
    // 在某个位置增加另外一个集合内所有元素
    public void add(int index, MyArrayList<E> c) {
        // check arguments
        // ...
        
        if (elementData.length < size + c.size()) {
            ensureCapacity(size + c.size());
        }
        // 先将index往后的元素往后挪，为新元素腾开空位。
        System.copyarray(...);
        // 再讲c里面的元素copy过来
        System.copyarray(...);
        size += c.size();
    }
    
    // 删除某个位置元素，如果产生不合法，是否要抛异常。
    // 函数返回值为void是否可以？
    public void remove(int index) {
        // check argument
        // ...
        
        // 将index及其后面的元素往前移动一个位置, memmove操作。
        // 从这里可以看到对于ArrayList，增加，删除操作是O(n)时间复杂度的。
        System.copyarray(...); 
        size -= 1;
    }
    
    // 从集合内删除其它集合内的元素。
    // ArrayList的实现，还有个boolean参数，控制开关，删除包含在c中的元素，还是删除不包含在c中的元素。
    // 可以看做为控制两个集合之差的形式是 A - B 还是 B - A。 
    public void remove(MyArrayList<E> c) {
        E[] newElementData = new E[elementData.length]
        int r = 0;
        int w = 0;
        for (int r = 0; r < size; ++r) {
            if (!c.contains(elementData[r])) {
                newElementData[w++] = elementData[r];
            }
        }
        elementData = newElementData;
        size = w;
    }
    
    // 修改某个位置元素值
    public set(int index, E e) {
        // check argument
        // ...
        elementData[index] = e;
    }
    
    // 查询
    // 如果index不合法，抛异常。
    public E get(int index) {
        // check argument
        // ...
        return elementData[index];
    }
    
    // 查询某个元素所在位置
    public int indexOf(E e) {
        for (int i = 0; i < size; ++i) {
            if (elementData[i].equals(e)) {
                return i;
            }
        }
        return -1;
    }
    
    // 判断某元素是否在表中
    public boolean contains(E e) {
        return indexOf(e) != -1;
    }
}
```

## 对比分析

将自己写的初步实现代码与JDK中的ArrayList做比较，对每一簇功能细致的比较、分析，搞清楚JDK1.8实现的机理。下面主要讨论下JDK中ArrayList的构造方法，它们与自己实现的preliminary版本区别在哪。

### ArrayList构造方法

#### ArrayList继承实现关系层次较深。

[![img](https://yangqisheng.github.io/images/ArrayList.png)](https://yangqisheng.github.io/images/ArrayList.png)

为什么要搞这么复杂的继承及实现关系呢，以后再说。先看几个问题：

1. ArrayList extends AbstractList implements List, AbstractList其实已经implements List了，为什么ArrayList还要implements List?

2. 看内部数组声明方式：transient Object[] elementData;
   a. 为什么用Object[]类型，为什么不用 E[]呢？
   b. 为什么用transient？
   c. 为什么用包访问权限？
   问题2.a，内部数组为什么要用Object[]，我没想明白。在stackoverflow上找到答案。Java里面不能创建泛型数组，编译通不过。看下面代码：

   ```
   public class Container<E> {
       E[] arr = new E[3]; // 这里会有编译错误，error generic array creation
   }
   ```

   为什么会这样？这和java的泛型实现有关，细节可以搜索相关资料。那不能用上面代码创建泛型数组，有其它方法吗？stackoverflow有人提出两种方法：

   ```
   // 方法1
   private Ojbect[] arr;
   arr = new Object[3];
   
   E get(int i) {
       return (E)arr[i]; // 每次获取元素都要转换类型，编译时会有unchecked的warning
   }
   
   // 方法2
   private E[] arr;
   arr = (E[]) new Object[3]; // 这里编译时会有unchecked的warning
   
   E get(int i) {
       return arr[i];
   }
   ```

   两者各有优缺点，java ArrayList是用的方法1。

   问题2.b，内部数组为什么要用transient？我们知道transient是禁止字段进行序列化，ArrayList的数据就是保存在elementData数组中的，前面用transient标识，那还怎么对ArrayList进行序列化啊，是不是有点奇怪？我们来想一想，ArrayList内部elementData多数时候是有空余空间的，如果直接用默认序列化方法，那么这些空余空间也会被序列化进去，显然这是不合适的。我们只需要序列化真正的数据部分。于是ArrayList采用的方法是禁用默认序列化方法，重写了序列化和反序列化方法writeObject和readObject方法。

   问题2.c，为什么用Default包访问权限，ArrayList自身的注释写着：

   ```
   transient Object[] elementData; // non-private to simplify nested class access
   ```

   简化嵌套类的访问。ArrayList内部有多个迭代器内部类，需要访问elementData。不过，经过我试验，Inner class是可以访问Outer class的private成员的。

3. 基于其它集合构建ArrayList，函数签名中的参数有点怪，那是干什么用的：

   ```
   public MyArrayList(Collection<? extends E> c);
   ```

   意思是可以接收E类型元素组成的集合，以及E的子类型元素组成的集合。可能有人会觉得，E的子类型集合可以自动转换成E类型集合，就像子类型自动可以转换成它的父类型。
   不可以的。看下面的例子：

   ```
   class Fruit {}
   class Apple extends Furit {}
   
   Collection<Furit> c = new Collection<>();
   c.add(new Fruit());
   c.add(new Apple()); // 这是可以的，Apple对象自动转换成Fruit。因为Apple "is-a" Fruit
   
   /*******************************************************/
   
   Collection<Fruit> cf = new Collection<>();
   cf.add(new Apple());
   
   Collection <Apple> ca = new Collection<>();
   ca.add(new Apple);
   
   // 假如有个函数接收集合作为参数。下面的定义只能接收Collection<Fruit>的参数，
   // 不能接收Collection<Apple>的参数。也就是说：虽然Apple是Fruit，
   // 但是“装apple的篮子不是装Fruit的篮子”。
   void func(Collection<Fruit> c) {
       // do somethig
   }
   func(cf); // 可以的。
   func(ca); // 不可以。因为 Collection<Apple> "is-not-a" Collection<Fruit>
   
   /*********有什么办法能让func也能接收Collection<Apple>********/
   // 这个函数参数的意思是：能接收所有Fruit及其子类组成的Collection。
   void func1(Collection<? extends Fruit>) {
       // do somethig
   }
   ```

   这里用到了Java泛型中的”通配符(Wildcards)“和”边界(Bounds)“概念。

- <? extends T> 是指”上界通配符“ (Upper Bounds Wildcards)
- <? super T> 是指”下界通配符“ (Lower Bounds Wildcards)
  细节可以搜索相关资料。

# 参考资料

stackoverflow https://stackoverflow.com/questions/13776576/why-does-the-arraylist-implementation-use-object
知乎 https://www.zhihu.com/question/20400700