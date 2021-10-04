---
type: docs
title: "CH39-FailFast"
linkTitle: "CH39-FailFast"
weight: 39
---

细心地朋友看Java容器源码时一定会发现在list()和listIterator()的注释中都有一句话：

> The iterators returned by this class’s iterator and listIterator methods are fail-fast.
> 

我看ArrayList源码没认真想fail-fast是什么意思，看Vector源码时又看到了这个词，而且在翻看Set实现类和Map实现类源码时也看到了这个词。fail-fast是什么？本篇文章以Vector为例来详细解说fail-fast。

## 什么是fail-fast

下面是Vector中源码的最上部的注释中关于fail-fast的介绍：

```
The iterators returned by this class's {@link #iterator() iterator} and
* {@link #listIterator(int) listIterator} methods are <em>fail-fast</em></a>:
* if the vector is structurally modified at any time after the iterator is
* created, in any way except through the iterator's own
* {@link ListIterator#remove() remove} or
* {@link ListIterator#add(Object) add} methods, the iterator will throw a
* {@link ConcurrentModificationException}.  Thus, in the face of
* concurrent modification, the iterator fails quickly and cleanly, rather
* than risking arbitrary, non-deterministic behavior at an undetermined
* time in the future.  The {@link Enumeration Enumerations} returned by
* the {@link #elements() elements} method are <em>not</em> fail-fast.
*
* <p>Note that the fail-fast behavior of an iterator cannot be guaranteed
* as it is, generally speaking, impossible to make any hard guarantees in the
* presence of unsynchronized concurrent modification.  Fail-fast iterators
* throw {@code ConcurrentModificationException} on a best-effort basis.
* Therefore, it would be wrong to write a program that depended on this
* exception for its correctness:  <i>the fail-fast behavior of iterators
* should be used only to detect bugs.</i>
```

由iterator()和listIterator()返回的迭代器是fail-fast的。在于程序在对list进行迭代时，某个线程对该collection在结构上对其做了修改，这时迭代器就会抛出ConcurrentModificationException异常信息。因此，面对并发的修改，迭代器快速而干净利落地失败，而不是在不确定的情况下冒险。由elements()返回的Enumerations不是fail-fast的。需要注意的是，迭代器的fail-fast并不能得到保证，它不能够保证一定出现该错误。一般来说，fail-fast会尽最大努力抛出ConcurrentModificationException异常。因此，为提高此类操作的正确性而编写一个依赖于此异常的程序是错误的做法，正确做法是：ConcurrentModificationException 应该仅用于检测 bug。

大意为在遍历一个集合时，当集合结构被修改，很有可能 会抛出Concurrent Modification Exception。为什么说是很有可能呢？从下文中我们可以知道，迭代器的remove操作（注意是迭代器的remove方法而不是集合的remove方法）修改集合结构就不会导致这个异常。

看到这里我们就明白了，fail-fast 机制是java容器（Collection和Map都存在fail-fast机制）中的一种错误机制。在遍历一个容器对象时，当容器结构被修改，很有可能会抛出ConcurrentModificationException，产生fail-fast。

## 什么时候会出现fail-fast？

在以下两种情况下会导致fail-fast，抛出ConcurrentModificationException

- 单线程环境：遍历一个集合过程中，集合结构被修改。注意，listIterator.remove()方法修改集合结构不会抛出这个异常。
- 多线程环境：当一个线程遍历集合过程中，而另一个线程对集合结构进行了修改。

单线程环境例子：

```java
import java.util.ListIterator;
import java.util.Vector;

public class Test {
/**
     * 单线程测试
     */
    @org.junit.Test
    public void test() {
        try {
            // 测试迭代器的remove方法修改集合结构会不会触发checkForComodification异常
            ItrRemoveTest();
            System.out.println("----分割线----");
            // 测试集合的remove方法修改集合结构会不会触发checkForComodification异常
            ListRemoveTest();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    // 测试迭代器的remove方法修改集合结构会不会触发checkForComodification异常
    private void ItrRemoveTest() {
        Vector list = new Vector<>();
        list.add("1");
        list.add("2");
        list.add("3");
        ListIterator itr = list.listIterator();
        while (itr.hasNext()) {
            System.out.println(itr.next());
            //迭代器的remove方法修改集合结构
            itr.remove();
        }
    }

    // 测试集合的remove方法修改集合结构会不会触发checkForComodification异常
    private void ListRemoveTest() {
        Vector list = new Vector<>();
        list.add("1");
        list.add("2");
        list.add("3");
        ListIterator itr = list.listIterator();
        while (itr.hasNext()) {
            System.out.println(itr.next());
            //集合的remove方法修改集合结构
            list.remove("3");
        }
    }
}
```

从结果中可以看到迭代器itr的remove操作并没有出现ConcurrentModificationException异常。而集合的remove操作则产生了异常。

多线程环境例子:

```java
import java.util.Iterator;
import java.util.List;
import java.util.ListIterator;
import java.util.Vector;

public class Test {
    private static List<String> list = new Vector<String>();

    /**
     * 多线程情况测试
     */
    @org.junit.Test
    public void test2() {
        list.add("1");
        list.add("2");
        list.add("3");
        // 同时启动两个线程对list进行操作！
        new ErgodicThread().start();
        new ModifyThread().start();
    }

    /**
     * 遍历集合的线程
     */
    private static class ErgodicThread extends Thread {
        public void run() {
            int i = 0;
            while (i < 10) {
                printAll();
                i++;
            }
        }
    }

    /**
     * 修改集合的线程
     */
    private static class ModifyThread extends Thread {
        public void run() {
            list.add(String.valueOf("5"));
        }
    }
    /**
     * 遍历集合
     */
    private static void printAll() {
        Iterator iter = list.iterator();
        while (iter.hasNext()) {
            System.out.print((String) iter.next() + ", ");
        }
        System.out.println();
    }
}
```

从结果中可以看出当一个线程遍历集合，而另一个线程对这个集合的结构进行了修改，确实有可能触发ConcurrentModificationException异常。

## fail-fast实现原理

下面是Vector中迭代器Itr的部分源码:

```java
/**
 * An optimized version of AbstractList.Itr
 */
private class Itr implements Iterator<E> {
    int expectedModCount = modCount;

    //省略的部分代码

    public void remove() {
        if (lastRet == -1)
            throw new IllegalStateException();
        synchronized (Vector.this) {
            checkForComodification();
            Vector.this.remove(lastRet);
            expectedModCount = modCount;
        }
        cursor = lastRet;
        lastRet = -1;
    }

    @Override
    public void forEachRemaining(Consumer<? super E> action) {
            //省略的部分代码
            checkForComodification();
        }
    }

    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
}
```

从代码中可以看到，每次初始化一个迭代器都会执行int expectedModCount = modCount;。modcount意为moderate count，即修改次数，对集合内容的修改都将增大这个值，如modCount++;。在迭代器初始化过程中会执行int expectedModCount = modCount;来记录迭会通过checkForComodification()方法判断modCount和expectedModCount 是否相等，如果不相等就表示已经有线程修改了集合结构。

使用迭代器的remove()方法修改集合结构不会触发ConcurrentModificationException，现在可以在源码中看出来是为什么。在remove()方法的最后会执行expectedModCount = modCount;，这样itr.remove操作后modCount和expectedModCount依然相等，就不会触发ConcurrentModificationException了。

## 如何避免fail-fast？

使用java.util.concurrent包下的类去取代java.util包下的类。所以，本例中只需要将Vector替换成java.util.concurrent包下对应的类即可。