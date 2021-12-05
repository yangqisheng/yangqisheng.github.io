---
title: Java-ArrayList-源码分析（二）
date: 2020-06-07
categories: Java
tags: [Java, ArrayList, 源码分析]
---

上一篇讲了ArrayList的几个构造函数。ArrayList的增删改查的实现，比较简单。本篇先讲一个代码效率问题，再着重讲讲ArrayList的迭代器实现。

<!--more-->

# 查询一个对象的索引位置

ArrayList的indexOf方法，接收一个Object，返回该Object在ArrayList中的位置。我们看下ArrayList的实现，如下：

```
// 写法一
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

我们可以看到，它是先判断o是否为null，根据结果分别走两个分支，在分支里面用for循环去挨个判断elementData中元素是否等于o。初步看，代码里面好像for循环写了两个，重复了，我们也可以在一个for循环里面判断，写成下面这种形式:

```
// 写法二
public int indexOf(Object o) {
    for (int i = 0; i < size; i++) {
        if (o == null) {
            if (elementData[i] == null)
                return i;
        } else {
            if (o.equals(elementData[i]))
                return i;
        }
    }
    return -1;
}
```

看起来，写法二比写法一少了一句for循环，但是写法二执行效率差一点，每次迭代都要判断一个o是否为null；而写法一只要判断一次o是否为null。所以总的下来，写法一节省了n/2 - 1次比较操作。

# 迭代器和内部类

ArrayList的迭代器是内部类实现的。迭代器是内部类的绝佳的应用场景。我们分几步来说明下：

- 为什么要迭代器，不用迭代器，可不可以对ArrayList进行迭代。
- 迭代器不用内部类实现，用独立的类实现会怎样。

## 为什么要迭代器

如果不用迭代器，也可以对ArrayList进行迭代。我们直接让ArrayList implements Iterator，增加一个成员变量来记录迭代的位置。这个时候ArrayList “is-a” Iterator。

```
class ArrayList<E> implements Iterator {
    private int next = 0; // index of the next element to return

    // 其它成员
    ...

   @Override
   boolean hasNext(); // 是否迭代到尾部

   @Override
   E next();          // 返回当前元素，并且后移迭代位置

   // 其它方法
   ...
}

/* client 对ArrayList进行迭代的时候是这样的 */
ArrayList a = new ArrayList()
// 对a进行了很多插入删除操作后
while (a.hasNext()) {
    doSomething(a.next());
}
```

这种实现方式会有什么问题？

- 只支持单向迭代。
- 一次只支持一个迭代。如果想同时进行两个迭代（比如一个快的，一个慢的），不支持。

## 不用内部类，用单独的类实现

假如用单独的ArrayListIterator类实现，那么ArrayListIterator类一定要关联一个ArrayList。如下：

```
class ArrayListIterator<E> implements Iterator {
    private int next;     // index of the next element to return
    private ArrayList<E> c; // 迭代器需要关联的集合

    <E> ArrayListIterator(ArrayList<E> c) {
        next = 0;
        this.c = c;
    }

    @Override
    boolean hasNext() {
        return next != c.size();
    }

    @Override
    E next() {
        return c.get(next++)
    }
}
```

你还可以实现一个逆向迭代器如下：

```
class ReverseArrayListIterator<E> implements Iterator {
    private int next;     // index of the next element to return
    private ArrayList<E> c; // 迭代器需要关联的集合

    <E> ReverseArrayListIterator(ArrayList<E> c) {
        next = c.size();
        this.c = c;
    }

    @Override
    boolean hasNext() {
        return next != -1;
    }

    @Override
    E next() {
        return c.get(next--);
    }
```

这种方式，貌似解决了之前的“只能单向迭代”以及一次只能用一个迭代器两个问题。但是产生了新的问题：

- Iterator中的每个方法都要访问及操作内部集合的内部数据，但是都是通过Collection的public方法，不能直接操作Collection内部数据，不仅效率低，而且访问权限受限制。
  这种实现方式，其实已经很接近内部类的实现方式了。内部类的实现方式解决了访问集合内部数据的限制问题。

## 看内部类的实现方法怎么表演

```
class ArrayList<E> {
    Object[] elementData;
    private int size;
    // 其它成员及方法
    ...

    class ListItr implements Iterator {
        private int next = 0;  // index of the next element to return

        @Override
        boolean hasNext() {
            return next == size; // 内部类可以直接访问外部类任何元素，包括成员和方法
        }

        @Override
        E next() {
            return elementData[next++];
        }
    }

    Iterator iterator() {
        return new ListItr();
    }
}

/* Client 调用代码*/
ArrayList<String> words = ArrayList<>();

// 插入删除一些String
...

Iterator itr = words.iterator();
while (itr.hasNext()) {
    doSomething(itr.next());
}
```

内部类实现的优势：

- 迭代器(内部类)可以直接访问集合(外部类)数据，效率高。
- 迭代器实现放在集合类内部，代码组织更好。
- 也可以在集合内部创建各种迭代器类。

为什么内部类能够无障碍访问外部类元素，因为内部类有个隐藏的引用，指向外部类对象。通过这个引用就可以访问到外部类对象的元素。

## 内部类杀手锏

> 如果没有内部类提供的、可以继承多个具体的或抽象的类的能力，一些设计与编程问题就很难解决。从这个角度看，内部类使得多重继承的解决方案变得完整。接口解决了部分问题，而内部类有效地实现了“多重继承”。也就是说，内部类允许继承多个非接口类型（类或抽象类）。

我们知道Java中没有多继承，只有单继承多implements。有人可能会说，可以implements多个接口，不就相当于多继承吗？非也。接口中不能有成员变量，也就是说一个类，只能从一个父类继承成员变量。那有什么办法能让达到多继承的效果呢？用内部类。

```
class A {
    private int i;
}

class B {
    private int j;
}

class C extends A {
    B makeB() {
        // 返回匿名内部类
        return new B() {

        }
    }
}

/* Client 代码*/
C c;
B b = c.makeB(); // 这个b可以看做B的一个匿名子类的对象，同时它的内部又有一个隐藏的指向C对象的引用。C又是A的子类。
```

通过这种方式，可以达到多继承的效果。细节可以参看 *Thinking in java On Java 8*

# 参考资料

Thinking in java on java 8