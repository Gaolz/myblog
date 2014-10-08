---
layout: post
title: "How the Hash Works in Ruby"
date: 2014-10-07 20:35:00
categories: ruby
tags: glz
author: "Gaolz"
---

## Ruby 里哈希(Hash) 是怎么工作的？
****  
> 原文请参考[这里](http://www.gotealeaf.com/blog/how-the-hash-works-in-ruby)
****  

一篇关于 Hash 数据结构的概述， 它在 Ruby 中是怎么实现的，窥视在 MRI Ruby 中 Hash 的演化历史。
****
### **什么是哈希(Hash)？**
哈希是一种数据结构，是以键值对的形式保存数据。字典,关联数组是它的别名。以键值对的形式保存这类关联的数据就可以进行快速的插入和查找，通常会控制在 O(1)

Ruby 中，可以通过声明字面量来创建一个哈希。

```ruby
h = {color: "black", font: "Monaco"}
h = {:color=>"black", :font=>"Monaco"}
```

或者通过 new 方法来创建:

```ruby
h = Hash.new
h[:color] = "black"
h[:font] = "Monaco"
```
****
### **哈希是怎么保存数据而他又为什么很高效**
要明白这个原理，让我们先看看基础的线性的数据结构，数组。数组允许我们随机的访问任何一个元素，如果我们事先知道该元素的索引。
```ruby
a = [1,2,4,5]
puts a[2] #> 4
```
如果我们要保存的键值对是一些限定在某范围内的整数，比如1-20或者1-100，这时用数组就行了，键是整数。
>比如，给出如下条件：
>> * 一个有20名学生的班，我们需要保存所有学生的姓名
>> * 每个学生有一个学生编号，在 1 到 20 之间
>> * 学生们的学生编号必须不同。

我们可以简单地通过数组保存他们的名字，就像下面的表所展示的那样：
|| *Key* || *Value*     ||
|| 1     || Belle       ||
|| 2     || Ariel       ||
|| 3     || Peter Pan   ||
|| 4     || Mickey Mouse||

```ruby
students= ['Belle', 'Ariel', 'Peter Pan', 'Mickey Mouse']
```

但是，如果当学生编号是4位数的形式呢？ 那我们就要分配一个包含10000个元素的表，通过编号访问名字。解决之道：4位数我们简化成只把最后面的两位数当键，所以我们可以随意访问数据。现在，若有个同学的编号是“3221”，以“21”结尾。我们就不得不保存两个值，而引发冲突。
|| *Key*      || *Hash(key) = last 2 digits* || *Value*      ||
|| 4221, 3221 || 21                          || Belle, Sofia ||
|| 1357       || 57                          || Ariel        ||
|| 4612       || 12                          || Peter Pan    ||
|| 1514       || 14                          || Mickey Mouse ||

```ruby
students = Array.new(100)
students[21]=['Belle', 'Sofia']
students[57]='Ariel'
students[12]='[Peter Pan]'
students[14]='Mickey Mouse'
```

如果学生编号是10位的字母数字形式的呢？照上面的方法来就会笨拙而低效。不幸的是，哈希可以轻松解决此题。

****

## *Ruby 的哈希是怎么工作的*

>所以，现在我们了解到哈希的目的是为了把一个给出的键转变为一个有限范围的整数。为了缩减范围，一个常用的技术是除法，所有的键被表的大小分成许多组，余数即为某条记录在表中的位置。因此，在上面的例子中，如果表的大小为20，那些记录的位置即为1， 17， 12， 14，下面的计算可以推导出来。

>> * 4221 % 20 = 1
>> * 1357 % 20 = 17
>> * 4612 % 20 = 12
>> * 1514 % 20

但是在真实的编程环境中，键并不总是规则的整数们，也有字符串，对象，或者其他的数据类型。这可以通过一个哈希的方法而不是键解决，应用除法得到正确的位置。该方法是一个数学方法，任意长度的一个字符串生成一个固定长度的整数值。哈希的名字来源于这种哈希机制。Ruby 用 [murmur hash function](http://en.wikipedia.org/wiki/MurmurHash) 并且通过和一个根据表的大小产生的基数 M 做除法这种机制实现哈希。

```ruby
murmur_hash(key) % M
```

上面一行的代码可以在 [Ruby](https://github.com/ruby/ruby/blob/1b5acebef2d447a3dbed6cf5e146fda74b81f10d/st.c) 语言的源代码中发现。

假如两个键指向同一个指，即所谓的哈希冲突，而值是链接到表中的同一位置或水桶的。
****

## *Ruby 是如何解决哈希冲突和增长的*

哈希里面面对的一个问题是分布问题。如果很多余数都落到同一位置(桶)处？那我们就必须首先找到表的量(通过计算键来算出)，然后检查链上的所有数据发现匹配的项。这样看起来创建哈希数据结构用来随意访问和缩减时间到 o(1) time 的目的就达不到了，因为现在我们不得不迭代所有的值去找寻记录，这样返回回来所需时间就会达到 O(n) time.

据发现如果除数 M 是基础的，结果不会太有偏差甚至是均匀的分布。但是，即使是最好的除数，冲突也难以避免正如记录的号码在不断地增加。Ruby 会基于密度来调整 M 的值。密度是表中在一个位置链上所有记录的数字。在上面的例子中，密度是2，因为我们在索引为1的位置处有2条记录。

Ruby 允许设置最大的密度的值为 5.

```ruby
#define ST_DEFAULT_MAX_DENSITY 5
```

当记录的密度达到5时，Ruby 会调整 M 的值，重新计算并且调整哈希中所有的记录。计算 M 的算法是生成接近于2的基数，来自于[Data Structures using C](http://www.amazon.com/Data-Structures-Using-Aaron-Tenenbaum/dp/0131997467).方法 new_size 的定义在原文[st.c](https://github.com/ruby/ruby/blob/1b5acebef2d447a3dbed6cf5e146fda74b81f10d/st.c)的第158行处。这里就是计算新的哈希大小的地方。

```c
new_size(st_index_t size)
{
  st_index_t i;
  for(i=3; i<31; i++) {
    if ((st_index_t)(1<<i) > size) return 1<<i;
  }
  return -1;
}
```

哈希在 JRuby 中的实现更容易阅读，要准备的基数是直接从一个整形数组中取出的。你可以看到下面的值是11，9如此等等。

```java
private static final int MRI_PRIMES[] = {
  8 + 3, 16 + 3, 32 + 5, 64 + 3, 128 + 3, 256 + 27, ....
};
```

这样当数据增加时重置可能会导致一个表现上的瑕疵，当哈希到达一个具体的大小时。Pat Shuaghnessy 做了一个详细的分析，在《Ruby under a Microscope》这本书中，当重置时你可以图形化数据，也才可以看到瑕疵。
****

## *Ruby 的哈希对于每一个进程都是是独一的*。

一件有趣的事这样说道，Ruby 里的每一个进程都是独一的。murmur 哈希通过随意的值来播种，每个 Ruby 的进程都会有一个不同的键得到一个不同的哈希。
****

Ruby 2.0 后 Ruby 包装哈希到了6条条目了。

另一个有趣的东西是 Ruby 的哈希非常小( <= 6)，并且在一个桶(位置)处被保存，而不会基于计算出来的键而传遍好几个位置。换言之，它们仅仅是一个值的数组(集).这是最近才被添加到源码中的。提交者做了如下批注。

“调查显示，通常的 rails 应用中小哈希被大量分配(使用)，达到了整个被使用的哈希量的40%而且从来没有超过一个元素的增长。”
****

## *Ruby 哈希键的排序*

自 Ruby 1.9.3 开始，哈希的一个属性是键可以基于它是如何被插入到哈希中来进行排序。一个有趣的问题来自 Ruby_forum, 问到支持这么做的原因以及它是怎样表现出高效的。上面有来自 Matz 的回答。下面即解释。

有人能解释为什么要添加这个特性吗？

在一些场景中有用，特别是关键字参数。

这样做会使得哈希里面的操作变慢吗？

不会，哈希涉及到的操作不会涉及到排序的信息，只是为了迭代器而已。内存也只是增加一点点消耗。

注意：关键字参数是在 Ruby 2.0 中添加进去的。下面有个例子

```ruby
def books(title: 'Programming Ruby')
  puts title
end

books # => 'Programming Ruby'
books(title: 'Eloquent Ruby')  # => 'Eloquent Ruby'
```
****

## *结论*

数据结构是通用的。一个哈希也是如此，不管它是在 Java, Python 还是在 Ruby 中实现的。有希望地，通过这个视角可以让我们看向远方，成为用户的哈希 API，提供对于我们每天所写的代码的效率更好的理解，甚至为下一升级版本的 Ruby 做出贡献。
****

## *参考资源*
1. [Data Structures using C](http://www.amazon.com/Data-Structures-Using-Aaron-Tenenbaum/dp/0131997467)
2. [Ruby Under a Microscope](http://patshaughnessy.net/ruby-under-a-microscope)
3. [st.c](https://github.com/ruby/ruby/blob/1b5acebef2d447a3dbed6cf5e146fda74b81f10d/st.c)
4. [RubyHash.java](https://github.com/jruby/jruby/blob/master/core/src/main/java/org/jruby/RubyHash.java)


