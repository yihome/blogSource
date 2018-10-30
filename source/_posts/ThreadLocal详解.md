---
title: ThreadLocal详解
date: 2018-07-19 10:18:26
tags: 
- Android
- 线程
categories: Android
---

## ThreadLocal是什么

`ThreadLocal`是java中处理并发问题的一种方式，但是和`Synchronized`、`volatile`等进程同步关键字不同，`ThreadLocal`主要用于进程隔离，即每一个线程都有一个自己的`ThreadLocal`，访问时访问的都是线程持有的对象，无法访问其他线程的`ThreadLocal`,也就不存在并发的问题。

## ThreadLocal使用

`ThreadLocal`使用起来类似一个数据封装类，使用set和get方法设置和获取存储的内容

```java
//创建ThreadLocal对象，使用泛型制定类型
ThreadLocal<Integer> localInt = new ThreadLocal<>();
//写入内容
localInt.set(10);
//获取内容
assert locaInt.get()==10;
```

`ThreadLocal`特殊的地方就在于不同线程间是无法获取到其它线程的相应对象的，在多线程场景中对ThreadLocal的使用实例如下

```java
ThreadLocal<Integer> localInt = new ThreadLocal<>();
Thread threadA = new Thread() {
    @Override
    public void run() {
        for(int i=100;i<200;i++){
            try {
                localInt.set(i);
                sleep(10);
                assert localInt.get()==i;
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        super.run();
    }
};
Thread threadB = new Thread() {
    @Override
    public void run() {
        for(int i=0;i<100;i++){
            try {
                localInt.set(i);
                sleep(10);
                assert localInt.get()==i;
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        super.run();
    }
};
threadA.start();
threadB.start();
```

ThreadA和ThreadB中`ThreadLocal`是不会相互影响的，他们每次从`ThreadLocal`中获取的都是线程内部的相应数据。

在上面的例子中我们只是用`ThreadLocal`存储了一个变量，如果想要同时拥有多个线程私有的变量就只能创建多个`TreadLocal`对象。

## ThreadLocal实现原理

`ThreadLocal`看上去很神奇，原理其实并不难，它只是把我们的变量关联到了线程的`Thread`对象上，我们借助`ThreadLocal`的set方法来说明这点。

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

在我们通过set方法设置`ThreadLocal`的内容是，它首先获取当前的`Thread`对象，同时获取`Thread`上的`ThreadLocalMap`，`ThreadLocalMap`事实上是hash表的一种实现，set事实上是把`ThreadLocal`对象和相应的value分别作为key和value保证在了这个map中，也就是说我们的value保存在Thread对象内。

![ThreadLocal](ThreadLocal.png)

get方法也是类似的，我们使用get方法获取值的时候，首先是获取currentThread，然后通过`Thread`的`ThreadLocalMap`用`ThreadLocal`作为key获取对应的value。

```java
 public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

如果没有创建过`ThreadLocalMap`，`setInitialValue`会创建`ThreadLocalMap`同时返回null。

## ThreadLocal内存泄漏

正常来说`ThreadLocal`的对象和其保存的value持有在一个普通的对象内，而在保存时他们会作为key和value保存在`Thread`对象的map内，线程一般拥有比普通对象更长的生命周期，特别是对于线程池中的线程。

这种情况下作为key的`ThreadLocal`和value就有泄漏的风险，`ThreadLocal`的设计上自然也考虑到了这点，因此`ThreadLocalMap`中Key都是作为弱引用存在的

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```
这样当持有`ThreadLocal`的对象销毁后，没有强引用的`ThreadLocal`也会很快被回收，但是与之对应的value却没法被自动回收。

事实上`ThreadLocalMap`对于这样的情况也是有所处理，即`ThreadLocalMap`在每次set或者get时发现有key为空的元素，都会把它清理掉

```java
 private void set(ThreadLocal<?> key, Object value) {
 	......
    for (Entry e = tab[i];
    e != null;
    e = tab[i = nextIndex(i, len)]) {
    	ThreadLocal<?> k = e.get();
        if (k == key) {
            e.value = value;
            return;
        }
		//清理掉key==null的元素
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
	}
	......
}
```
但是即使这样光靠`ThreadLocalMap`也没有办法完全避免value的内存泄漏，最好的就是每次使用完`ThreadLocal`后都调用它的remove方法，清除数据。