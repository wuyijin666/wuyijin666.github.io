---
title: 浅谈Threadlocal（1）
date: 2024-04-13 23:50:28
tags:  浅谈Threadlocal（1）
categories: 多线程与并发
---
### ThreadLocal简介
#### ThreadLocal <mark> (线程变量)</mark>，意思是ThreadLocal中填充的变量属于当前线程。该变量对其他线而言是隔离的，该变量是当前线程的独有变量。ThreadLocal为变量在每个线程中都创建了一个副本，那么每个线程可以访问自己内部的副本变量。

***它的使用场合主要是为了解决多线程中因为数据并发产生不一致问题。***

ThreadLoal 变量，线程局部变量，同一个 ThreadLocal 所包含的对象，在不同的 Thread 中有不同的副本。这里有几点需要注意：

1. 因为每个 Thread 内有自己的实例副本，且<mark>该副本只能由当前 Thread 使用</mark>。这是也是 ThreadLocal 命名的由来。
2. 既然每个 Thread 有自己的实例副本，且其它 Thread 不可访问，那就<mark>不存在多线程间共享的问题</mark>。
3. ThreadLocal 提供了线程本地的实例。它与普通变量的区别在于，每个使用该变量的线程都会初始化一个完全独立的实例副本。ThreadLocal 变量通常被private static修饰。当一个线程结束时，它所使用的所有 ThreadLocal 相对的实例副本都可被回收。
#### 总的来说，ThreadLocal 适用于每个线程需要自己独立的实例且该实例需要在多个方法中被使用，也即变量在线程间隔离而在方法或类间共享的场景.
下图是ThreadLocal在使用过程中的状态：
![](/images/1.png)
### ThreadLocal与Synchronized的区别
ThreadLocal<T>其实是与线程绑定的一个变量。ThreadLocal和Synchonized都用于解决多线程并发访问。

ThreadLocal与Synchronized本质区别在于： 

1. Synchronized用于线程间的数据共享，而ThreadLocal用于线程间的数据隔离。
2. Synchronized利用锁机制，使变量或代码块在某一时刻只能被一个线程访问。而ThreadLocal为每个线程都提供了变量的副本，使得每个线程在某一时间访问到的并不是同一个对象，这样就隔离了多个线程对数据的共享。而Synchronized却正好相反，用于在多个线程间通信是数据共享。

***一句话理解ThreadLocal，threadlocl是作为当前线程中属性ThreadLocalMap集合中的某一个Entry的key值Entry（threadlocl,value），虽然不同的线程之间threadlocal这个key值是一样，但是不同的线程所拥有的ThreadLocalMap是独一无二的，也就是不同的线程间同一个ThreadLocal（key）对应存储的值(value)不一样，从而到达了线程间变量隔离的目的，但是在同一个线程中这个value变量地址是一样的。***

### ThreadLocal的简单使用
```java
public class ThreadLocaDemo {
 
    private static ThreadLocal<String> localVar = new ThreadLocal<String>();
 
    static void print(String str) {
        //打印当前线程中本地内存中本地变量的值
        System.out.println(str + " :" + localVar.get());
        //清除本地内存中的本地变量
        localVar.remove();
    }
    public static void main(String[] args) throws InterruptedException {
 
        new Thread(new Runnable() {
            public void run() {
                ThreadLocaDemo.localVar.set("local_A");
                print("A");
                //打印本地变量
                System.out.println("after remove : " + localVar.get());
               
            }
        },"A").start();
 
        Thread.sleep(1000);
 
        new Thread(new Runnable() {
            public void run() {
                ThreadLocaDemo.localVar.set("local_B");
                print("B");
                System.out.println("after remove : " + localVar.get());
              
            }
        },"B").start();
    }
}
 
A :local_A
after remove : null
B :local_B
after remove : null
 
```
从该示例中，两个线程分别获取自己线程存放的变量，变量获取不会错乱。





