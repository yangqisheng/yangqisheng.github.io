---
title: Python中的iterable和iterator
date: 2020-06-07
categories: Python
tags: [Python, Iterable, Iterator]
---

Python编程中，我们经常会遇到iterable和iterator的概念，二者之间既有区别又有联系。这篇博文就来详细说说这个话题。

<!--more-->

## 说在前面

迭代器在各种编程语言中都是一个重要的概念。它是对一个对象或数据集合进行遍历的一种抽象。比如一个列表，一个字典，如果需要挨个打印里面的元素，我们就可以针对它们创建一个迭代器，然后就可以很方便的进行遍历了。

## Python的iterable和iterator

Python中除了iterator对象还有iterable对象。我们知道iterator是迭代器，那iterable是个什么东西呢，它是可迭代对象。什么是可迭代对象？我理解就是可以利用迭代器对其进行遍历的东西。
Python中规定，只要类实现了__iter__方法并且该方法返回一个iterator的class就是iterable。

## iterable

Python中的list, dict,tuple等基本类型都是iterable，执行下面的python代码可以看到list对象有__iter__方法。

```
dir([1, 2, 3, 4,])
```

下面我们构建一个自己的iterable,设定这个iterable基于一个字符串构建，切词形成一个可迭代的word集合，然后可以对里面的word进行迭代。

```
class MyIteratable:
    def __init__(self, string):
        self.string = string

    # 这里要注意，返回一个iterator对象，利用这个iterator可以对该Myiterable进行迭代。
    def __iter__(self):
        return MyIterator(self.string)

class MyIterator:
    def __init__(self, string):
        pass
    def __next__(self):
        pass
```

client代码如下:

```
my_iterable = MyIteratable("It's a test")

# iter(my_iterable)会自动调用my_iterable.__iter__方法。python类中带有双下划线前后缀的方法叫Magic Methods, 类似的还有__init__, __next__, __repr__等等。
my_iterator = iter(my_iterable)

# 下面就可以用my_iterator进行迭代了，具体怎么使用iterator进行迭代请看下面
...
```

## iterator

Python中只要实现了__next__方法的class叫iterator（严格来说，还需要实现__iter__，这个后面再讲）。每调用一次next(iterator), 就取出了一个元素。当遍历完最后一个元素后，还调用next(iterator)就会触发StopIteration异常。所以我们利用一个iterator进行迭代时，就有client代码如下:

```
# 通过iter(iterable)或者其它方法获取一个迭代器
my_iterator = iter(iterable)
while True:
    try:
        item = next(my_iterator) # 获取到一个元素
    except StopIteration:
        break
```

next(my_iterator)其实是调用了my_iterator.__next__方法。这个方法每次都需要返回一个元素。接下来，我们来完善上面class MyIterator里面的方法。__next__方法每次返回一个元素，当到达尾部时触发StopIteration异常。

```
class MyIteratable:
    def __init__(self, string):
        self.string = string

    # 这里要注意，返回一个iterator对象，利用这个iterator可以对该Myiterable进行迭代。
    def __iter__(self):
        return MyIterator(self.string)

class MyIterator:
    def __init__(self, string):
        self.words = string.split()
        self.index = 0 # 每次迭代，迭代器都有个位置。

    def __next__(self):
        if self.index < len(self.words):
            item = self.words[self.index]
            self.index += 1
            return item
        else:
            raise StopIteration()
```

再次回到上面使用iterator迭代一个iterable的client代码，先要获取iterable的iterator，然后在while循环里面遍历元素，while里面退出条件是捕获到StopIteration异常。我们可以看到，对一个iterable进行迭代太啰嗦了，那么能否有个简洁的方法呢？有！请看下面：

```
for i in iterable:
    print(i)
```

for in 语法就是对上面啰嗦写法的一个简写形式。for in是语言内部进行了上面那些啰嗦的操作。
好了，到现在，我们基本捋清了iterable和iterator的区别和联系了吧。总结如下：

- iterable是可迭代对象，实现了__iter__方法，该方法返回一个可以对自身进行迭代的迭代器。
- iterator是迭代器对象，实现了__next__方法，可以对该迭代器所关联的iterable进行迭代。
- iterable内部元素数量是可知的，而我们不能通过iterator知道其关联的iterable有多少数量，只能一步一步的next，最后触发StopIteration。

## 重要的一点, iterator也是一个iterable。

iterator还需要实现__iter__方法，而且该方法很简单就是返回自身，如下：

```
class MyIterator:
    def __init__(self, string):
        self.words = string.split()
        self.index = 0 # 每次迭代，迭代器都有个位置。

    def __next__(self):
        if self.index < len(self.words):
            item = self.words(self.index)
            self.index += 1
            return item
        else:
            raise StopIteration()

    def __iter__(self):
        return self
```

iterator为什么要实现__iter__，而且大多数情况下都是返回self。是不是看起来很奇怪？如果没有这个__iter__函数，会不会有问题？这个问题我也是找了好多资料才想明白的。
情况是这样的，我们知道一个iterable可以应用在for in 语句中，更进一步，一个iterator也是可以应用在for in 语句中的。当应用在for in 中时，iterator就当做一个iterable使用了。

```
my_list = [1, 2, 3, 4]
my_list_iter = iter(my_list)
for i in my_list_iter:
    print(i)
```

__iter__必须返回一个迭代器，而一个iterator的__iter__函数，它要返回一个迭代器，必然是它自身。

## 参考资料

- Python 3 Object-oriented Programming, 2nd Edition