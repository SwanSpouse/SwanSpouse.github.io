---
title: java 单例模式
date: 2018-09-06 23:58:08
tags:
  - java
  - 设计模式
---

### 单例模式定义

Ensure a class has only one instance, and provide a global point of access to it.（确保某一个类只有一个实例，而且自行实例化并向整个系统提供这个实例。）

单例的实现主要是通过以下两个步骤：

* *将构造方法定义为私有方法*，这样其他代码就无法通过调用该类的构造方法来实例化该类的对象，只有通过该类提供的静态方法来得到该类的唯一实例；
* *在该类内提供一个静态方法*，当调用这个方法时，如果类持有的引用不为空就返回这个引用；如果类持有的引用为空就创建该类的实例并将实例的引用赋值给类持有的引用。


### 单例模式的5种写法

写单例模式的时候应该注意的问题：
* 是否是lazy loading（延迟加载）
* 是否是线程安全的
* 保证线程安全的同时是否也兼顾了效率问题

理解饿汉、懒汉。要从其角色的角度出发来进行考虑。
* 饿汉，最希望的就是获得（饿），所以总是迫不及待的做某事，尽可能的提前实例化的时间。
* 懒汉，最希望的就是不做（懒），所以总是躲避着做某事，尽可能的拖延实例化的时间。

#### 饿汉模式

```java

public class Singleton {

    private final static Singleton INSTANCE = new Singleton();

    private Singleton () {}

    public static Singleton getInstance() {
        return INSTANCE;
    }
}

```

优点：这种写法比较简单，就是在加载的时候就完成实例化。避免了线程同步问题。
缺点：在类加载的时候就完成实例化，没有达到lazy loading的效果。如果从始至终从未使用过这个实例，则会造成内存的浪费。


#### 懒汉模式

```java
public class Singleton {

    private final static Singleton INSTANCE;

    private Singleton () {}

    public static Singleton getInstance() {
        if (INSTANCE == null) {
            INSTANCE = new Singleton();
        }
        return INSTANCE;
    }
}

```

这种写法起到了lazy loading的效果，但是只能在单线程下使用。如果在多线程下，一个线程进入了 if(INSTANCE == null)的判断，同时另一个线程也进入了此判断语句，此时便会产生多个实例。

Q: 即使创建了多个实例，这种方法会造成线程不安全吗？ 感觉创建多个也无妨，正常使用就好了。或者说比较极端的例子，N个线程会同时创建N个实例，造成资源浪费？

#### 懒汉模式 + 线程安全（同步方法）

解决上述问题的最简单办法就是使用synchronized关键字。每个线程在想获得类的实例时候，执行getInstance()方法都要进行同步。导致效率非常低。


```java
public class Singleton {
    private static Singleton INSTANCE;

    private Singleton() {}

    public static synchronized Singleton getInstance() {
        if(INSTANCE == null) {
            INSTANCE = new Singleton();
        }
        return INSTANCE;
    }  
}
```


#### 懒汉模式 + 线程安全（双重校验）

Double-Check，进行了两次if(INSTANCE == null)检查，这样就可以保证线程安全了。这样，实例化代码只用执行一次，后面再次访问时，判断if(INSTANCE == null)，直接return实例化对象。

```java
public class Singleton {
    private static Singleton INSTANCE;

    private Singleton() {}

    public static Singleton getInstance() {
        if(INSTANCE == null) {
            synchronized (Singleton.class) {
                if(INSTANCE == null) {
                    INSTANCE = new Singleton();
                }
            }
        }
        return INSTANCE;
    }  
}
```

Double-Check的精髓在于先判断 INSTANCE == null 然后决定是否加锁；加锁之后，仍需再次判断 INSTANCE == null ,决定是否实例化。

1. 第一个判断是为了提高效率，使不用加锁的大多数情况直接返回。
2. 第二个判断是为了线程安全，防止在多线程情况下产生多个实例。

#### 静态内部类

```java
public class Singleton {

  private Singleton() {}

    private static class SingletonInstance {
        private static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return SingletonInstance.INSTANCE;
    }
}
```

这种方式跟饿汉式方式采用的机制类似，但又有不同。两者都是采用了类装载的机制来保证初始化实例时只有一个线程。不同的地方在饿汉式方式是只要Singleton类被装载就会实例化，没有Lazy-Loading的作用，而静态内部类方式在Singleton类被装载时并不会立即实例化，而是在需要实例化时，调用getInstance方法，才会装载SingletonInstance类，从而完成Singleton的实例化。

类的静态属性只会在第一次加载类的时候初始化，所以在这里，JVM帮助我们保证了线程的安全性，在类进行初始化时，别的线程是无法进入的。

优点：避免了线程不安全，延迟加载，效率高。

### reference

* https://www.cnblogs.com/zhaoyan001/p/6365064.html
