---
title: 浅谈Threadlocal（2）（源码分析）
date: 2024-04-14 12:18:28
tags:  浅谈Threadlocal（2）
categories: 多线程与并发
---
### ThreadLocal原理（源码分析）
1. ThreadLocal的set方法：
```java
public void set(T value){
    //1.获取当前线程
    Thread t = Thread.currentThread();
    //2. 获取线程中的属性threadLocalMap,若threadLocalMap不为空，则直接更新要保存的变量值；
    //否则创建threadLocalMap，并赋值。
    ThreadLocalMap map = getMap(t);
    if(map != null){
        map.set(this, value);
    }else{
      //初始化threadLocalMap 并赋值
      creatMap(t, value);
    }
}
``` 
从上面代码可知，ThreadLocal set赋值时首先会获取当前线程thread,并获取thread线程中的threadLocalMap属性。如果map属性不为空，则直接更新value值，如果为空，则实例化threadLocalMap，并赋值。

接下来分析一下 threadLocalMap 和 createMap 的作用：
```java
  static class ThreadLocalMap {
 
        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;
 
            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
 
        
    }
```
可以看出 ThreadLocalMap是 ThreadLocal的静态类，它的构成主要用Entry来保存数据，而且还是继承的弱引用。在Entry内部使用ThreadLocal作为key，使用我们设置的value作为value。
```java
//这个是threadlocal 的内部方法
void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
 
 
    //ThreadLocalMap 构造方法
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
``` 
2. ThreadLocal的get方法：
 ```java
 public T get(){
    //1. 获取当前线程
    Thread t = Thread.currentThread();
    //2. 获取当前线程的ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    //3. 如果map不为空
    if(map != null){
        //获取ThreadLocalMap中存储的值
        ThreadLocalMap.Entry e = map.getEntry(this);
        if(e != null){
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    } 
    //如果数据为null,则初始化，初始化结果，ThreadLocalMap中存放key值为ThreadLocal,值为null
    return setInitialValue();

    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
 }
 ```
 3. ThreadLocal的remove方法：
```java
public void remove(){
    ThreadLocalMap m = getMap(Thread.currentThread());
    if(m != null){
        m.remove(this);
    }
}
```
**remove方法，直接将ThreadLocal对应的值从当前线程Thread中ThreadLocalMap中删除。为何删除，这涉及内存泄漏 （memory leak，指程序在申请内存后，无法释放已申请的内存空间，一次内存泄露危害可以忽略，但内存泄露堆积后果很严重，无论多少内存,迟早会被占光。）的问题。**
 实际上 ThreadLocalMap 中使用的 key 为 ThreadLocal 的弱引用，弱引用的特点是，如果这个对象只存在弱引用，那么在下一次垃圾回收的时候必然会被清理掉。

所以如果 ThreadLocal 没有被外部强引用的情况下，在垃圾回收的时候会被清理掉的，这样一来 ThreadLocalMap中使用这个 ThreadLocal 的 key 也会被清理掉。但是，value 是强引用，不会被清理，这样一来就会出现 key 为 null 的 value。

ThreadLocal其实是与线程绑定的一个变量，如此就会出现一个问题：如果没有将ThreadLocal内的变量删除（remove）或替换，它的生命周期将会与线程共存。通常线程池中对线程管理都是采用线程复用的方法，在线程池中线程很难结束甚至于永远不会结束，这将意味着线程持续的时间将不可预测，甚至与JVM的生命周期一致。举个例字，如果ThreadLocal中直接或间接包装了集合类或复杂对象，每次在同一个ThreadLocal中取出对象后，再对内容做操作，那么内部的集合类和复杂对象所占用的空间可能会开始持续膨胀。
#### ThreadLocal与Thread，ThreadLocalMap之间的关系
![](/images/3.png)
![](/images/4.png)
从这个图中我们可以非常直观的看出，ThreadLocalMap其实是Thread线程的一个属性值，而ThreadLocal是维护ThreadLocalMap这个属性值的一个工具类。<mark>Thread线程可以拥有多个ThreadLocal维护的自己线程独享的共享变量</mark>（这个共享变量只是针对自己线程里面共享）

   
   