---
title: ThreadLocal（3）常见使用场景
date: 2024-04-14 20:50:28
tags:  浅谈Threadlocal（3）常见使用场景
categories: 多线程与并发
---
### ThreadLocal 常见使用场景
ThreadLocal作用在每一个线程内都需要独立的保存信息，这样就方便同一个线程的其他方法获取到该信息的场景，由于每一个线程获取到的信息可能都不一样，前面执行的方法保存了信息后，后续方法可以通过ThreadLocal直接获取到，避免了传参，类似于全局变量的概念。比如像用户登录令牌解密后的信息传递、用户权限信息、从用户系统中获取到的用户名。
![](/images/15.png)
如上图，线程创建了变量A,方法二是跟方法一在同一个线程内，变量A在该线程中共享。
```java
//用户微服务配置token解密信息传递例子
public static ThreadLocal(LoginUser) threadLocal = new ThreadLocal<>();
     LoginUser loginUser = new LoginUser();
     loginUser.setId(id);
     loginUser.setName(name);
     loginUser.setMail(mail);
     loginUser.setHeadImg(headImg);
     threadLocal.set(loginUser);

后续想直接获取到，就threadLocal.getXXX即可
```
### 如何用ThreadLocal实现线程安全的问题
ThreadLocal适用于如下两种场景：
1. 每个线程需要自己单独的实例
2. 实例需要在多个方法中共享，但不希望被多线程共享

## 应用场景
1. 存储用户Session
   
