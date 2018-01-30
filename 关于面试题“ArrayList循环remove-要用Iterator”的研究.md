---
title: 关于面试题“ArrayList循环remove()要用Iterator”的研究
date: 2017-10-24 15:05:54
tags:
---
两个月前我在参加一场面试的时候，被问到了`ArrayList`如何循环删除元素，当时我回答用`Iterator`,当面试官问为什么要用`Iterator`而不用`foreach`时，我没有答出来，如今又回想到了这个问题，我觉得应该把它搞一搞，所以我就写了一个小的demo并结合阅读源代码来验证了一下。

下面是我验证的`ArrayList`循环remove()的4种情况,以及其结果(基于oracle jdk1.8)：

```
//List<Integer> list = new ArrayList<>();
//list.add(1);
//list.add(2);
//list.add(3);
//list.add(4);
//循环remove()的4种情况的代码片段：

//#1
for (Integer integer : list) {
    list.remove(integer);
}

结果：
Exception in thread "main" java.util.ConcurrentModificationException
	at java.util.ArrayList$Itr.checkForComodification(ArrayList.java:901)
	at java.util.ArrayList$Itr.next(ArrayList.java:851)
-----------------------------------------------------------------------------------

//#2
Iterator<Integer> iterator = list.iterator();
while(iterator.hasNext()) {
    Integer integer = iterator.next();
    list.remove(integer);
}

结果：
Exception in thread "main" java.util.ConcurrentModificationException
	at java.util.ArrayList$Itr.checkForComodification(ArrayList.java:901)
	at java.util.ArrayList$Itr.next(ArrayList.java:851)
-----------------------------------------------------------------------------------


//#3
for (int i = 0; i < list.size(); i++) {
    list.remove(i);
}
System.out.println(list);

结果：
[2, 4]
-----------------------------------------------------------------------------------

//#4
Iterator<Integer> iterator = list.iterator();
while (iterator.hasNext()){
    iterator.next();
    iterator.remove();
}
System.out.println(list.size());

结果：(唯一一个得到期望值的)
0
```
可以看出来这几种情况只有最后一种是得到预期结果的，其他的要么异常要么得不到预期结果，下面咱们一个一个进行分析。

### #1
```
//#1
for (Integer integer : list) {
    list.remove(integer);
}

结果：
Exception in thread "main" java.util.ConcurrentModificationException
	at java.util.ArrayList$Itr.checkForComodification(ArrayList.java:901)
	at java.util.ArrayList$Itr.next(ArrayList.java:851)
```
通过异常栈，我们可以定位是在`ArrayList`的内部类`Itr`的`checkForComodification`方法中爆出了`ConcurrentModificationException`异常(关于这个异常是怎么回事咱们暂且不提)我们打开`ArrayList`的源码，定位到901行处：
```
final void checkForComodification() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
}
```
这个爆出异常的方法实际上就做了一件事，检查`modCount != expectedModCount`因为满足了这个条件，所以抛出了异常，继续查看`modCount`和`expectedModCount`这两个变量，发现`modCount`是继承自`AbstractList`的一个属性，这个属性有一大段注释
```
/**
 * The number of times this list has been <i>structurally modified</i>.
 * Structural modifications are those that change the size of the
 * list, or otherwise perturb it in such a fashion that iterations in
 * progress may yield incorrect results.
 *
 * <p>This field is used by the iterator and list iterator implementation
 * returned by the {@code iterator} and {@code listIterator} methods.
 * If the value of this field changes unexpectedly, the iterator (or list
 * iterator) will throw a {@code ConcurrentModificationException} in
 * response to the {@code next}, {@code remove}, {@code previous},
 * {@code set} or {@code add} operations.  This provides
 * <i>fail-fast</i> behavior, rather than non-deterministic behavior in
 * the face of concurrent modification during iteration.
 *
 * <p><b>Use of this field by subclasses is optional.</b> If a subclass
 * wishes to provide fail-fast iterators (and list iterators), then it
 * merely has to increment this field in its {@code add(int, E)} and
 * {@code remove(int)} methods (and any other methods that it overrides
 * that result in structural modifications to the list).  A single call to
 * {@code add(int, E)} or {@code remove(int)} must add no more than
 * one to this field, or the iterators (and list iterators) will throw
 * bogus {@code ConcurrentModificationExceptions}.  If an implementation
 * does not wish to provide fail-fast iterators, this field may be
 * ignored.
 */
protected transient int modCount = 0;
```
大致的意思是这个字段用于有`fail-fast`行为的子集合类的，用来记录集合被修改过的次数，我们回到`ArrayList`可以找到在`add(E e)`的调用链中的一个方法`ensureExplicitCapacity(int minCapacity)` 中会对`modCount`自增：
```
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;
    
    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}
```
我们在初始化list时调用了4次`add(E e)`所以现在`modCount`的值为`4`

{% asset_img 2da04a057d30f7e79a6ee8f1f7ecc083.png %}
再来找`expectedModCount`：这个变量是定义在`ArrayList`的`Iterator`的实现类`Itr`中的，它默认被赋值为`modCount`

{% asset_img 37b0a74536d90e1293623b60874d4809.png %}
知道了这两个变量是什么了以后，我们开始走查吧，在`Itr`的相关方法中加好断点(编译器会将`foreach`编译为使用`Iterator`的方式，所以我们看`Itr`就可以了)，开始调试：

循环：
{% asset_img b3909c53a5b18446e517c8e39b602e89.png %}
在迭代的每次`next()`时都会调用`checkForComodification()`
{% asset_img 4897e7f3c5f022ce2e38d8fd1f96c300.png %}
`list.remove()`：
{% asset_img 62a566cf28a6b0491a55a9662f4b34c1.png %}
在`ArrayList`的`remove(Object o)`中又调用了`fastRemove(index)`：

{% asset_img 3d161f02e988e0299239303aa4667f6f.png %}
`fastRemove(index)`中对` modCount`进行了自增,刚才说过` modCount`经过4次`add(E e)`初始化后是`4`所以`++`后现在是`5`：

{% asset_img d1e22bb73926e7e1fd64b80ea00be35a.png %}


继续往下走，进入下次迭代：

{% asset_img 95cdf560ba5ed016628b5a84e5151745.png %}
又一次执行`next()`，`next()`调用`checkForComodification()`，这时在上边的过程中` modCount`由于`fastRemove(index)`的操作已经变成了`5`而`expectedModCount`则没有人动，所以很快就满足了抛出异常的条件`modCount != expectedModCount`(也就是前面提到的`fail-fast`)，程序退出。

{% asset_img 73dc584d91d1bda2497722076cfc1938.png %}

{% asset_img c62b46700f8153a041f420edaf2757ee.png %}

{% asset_img 15665e525143d1bb3f01f2c3f92d32d6.png %}

### #2
```
//#2
Iterator<Integer> iterator = list.iterator();
while(iterator.hasNext()) {
    Integer integer = iterator.next();
    list.remove(integer);
}

结果：
Exception in thread "main" java.util.ConcurrentModificationException
	at java.util.ArrayList$Itr.checkForComodification(ArrayList.java:901)
	at java.util.ArrayList$Itr.next(ArrayList.java:851)
```
其实这个#2和#1是一样的，`foreach`会在编译期被优化为`Iterator`调用，所以看#1就好啦。

{% asset_img f37356b67a12596f9a3bc84940d4d0d7.png %}
{% asset_img 66f6758247a10b3d43b5aea0ea7c4056.png %}

### #3
```
//#3
for (int i = 0; i < list.size(); i++) {
    list.remove(i);
}
System.out.println(list);

结果：
[2, 4]
```
这种一本正经的胡说八道的情况也许在写代码犯困的情况下会出现... 不做文字解释了，用`println()`来说明吧：


{% asset_img b2d629e60abe61dacc71b18d6676a824.png %}
```
第0次循环开始
remove(0)前的list: [1, 2, 3, 4]
remove(0)前的list.size()=4
执行了remove(0)
remove(0)后的list.size()=3
remove(0)后的list: [2, 3, 4]
下一次循环的i=1
下一次循环的list.size()=3
第0次循环结束
是否还有条件进入下次循环?: true

第1次循环开始
remove(1)前的list: [2, 3, 4]
remove(1)前的list.size()=3
执行了remove(1)
remove(1)后的list.size()=2
remove(1)后的list: [2, 4]
下一次循环的i=2
下一次循环的list.size()=2
第1次循环结束
是否还有条件进入下次循环?: false


Process finished with exit code 0
```
实际上`ArrayList`中`Itr`用`游标`和`最后一次返回值索引`来解决了这种`size`越删越小，但是要删除元素的index越来越大的尴尬局面，这个将在#4里说明。

### #4
这个才是正儿八经能够正确执行的方式，用了`ArrayList`中迭代器`Itr`的`remove()`而不是用`ArrayList`本身的`remove()`，我们调试一下吧看看到底经历了什么：

迭代：

{% asset_img 22ae661ec82e1804cd2510e325c71612.png %}

`Itr`初始化：游标 cursor = 0; 最后一次返回值索引 lastRet = -1; 期望修改次数 expectedModCount = modCount = 4;


{% asset_img a18ded7b54e714ed93ab937ea49aecf5.png %}
迭代的`hasNext()`：检查游标是否已经到达当前list的`size`，如果没有则说明可以继续迭代：

{% asset_img cf4046a207c36956a55ce54e78f92b49.png %}
{% asset_img 53d691f89d85956b22f1113bab80b0bc.png %}
迭代的`next()`： `checkForComodification()` 此时`expectedModCount`和`modCount`是相等的，不会抛出`ConcurrentModificationException`，然后取到游标(第一次迭代游标是`0`)对应的list的元素，再将游标+1，也就是游标后移指向下一个元素，然后将游标原值`0`赋给最后一次返回值索引，也就是最后一次返回的是索引`0`对应的元素

{% asset_img 9d5fdf583b8f2c7c49a276006848f0af.png %}

{% asset_img 03884bb2f5c8fd4824269d4e59ee0832.png %}

`iterator.remove()`：同样`checkForComodification()`然后调用`ArrayList`的`remove(lastRet)`删除最后返回的元素,删除后`modCount`会自增

{% asset_img 73197144824a16a7df6de501f3f1ceaa.png %}

{% asset_img eb0335095275795bfabdb4afeab664a5.png %}

删除完成后，将游标赋值成最后一次返回值索引，其实也就是将游标回退了一格回到了上一次的位置，然后将最后一次返回值索引重新设置为了初始值`-1`,最后`expectedModCount`又重新赋值为了上一步过程完成后新的`modCount`


{% asset_img 3994ac58eb61923481696ee5d19b63a3.png %}

{% asset_img c348c539f5cd8d572c9c3cf7a8decc2b.png %}
由上两个步骤可以看出来，虽然list的`size`每次`remove()`都会`-1`，但是由于每次`remove()`都会将游标回退，然后将最后一次返回值索引重置，所以实际上没回`remove()`的都是当前集合的第`0`个元素，就不会出现#3中`size`越删越小，而要删除元素的索引越来越大的情况了，同时由于在`remove()`过程中`expectedModCount`和`modCount`始终通过赋值保持相等，所以也不会出现`fail-fast`抛出异常的情况了。

以上是我通过走查源码的方式对面试题“ArrayList循环remove()要用Iterator”做的一点研究，没考虑并发场景，这篇文章写了大概3个多小时，写完这篇文章办公室就剩我一个人了，我也该回去了，今天1024程序员节，大家节日快乐！


---
### 2017.10.25更新#1

感谢`@llearn`的提醒，#3也可以用用巧妙的方式来得到正确的结果的(再面试的时候，我觉得可以和面试官说不一定要用`Iterator`了，感谢`@llearn`：

>//#3 我觉得可以这样
for (int i = 0; i < list.size(); ) {
    list.remove(0);
}
System.out.println(list);

### 2017.10.25更新#2
感谢`@ChinLong`的提醒，提供了另一种不用`Iterator`的方法，也就是倒着循环(这种方案我写完文章时也想到了，但没有自己印证到demo上)，感谢`@ChinLong`：

>然道就没有人和我一下喜欢倒着删的.
听别人说倒着迭代速度稍微快一点???
for (int i = list.size() -1;  i >= 0; i-- ) {
  list.remove(i);
}
System.out.println(list);