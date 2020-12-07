#### 1. 什么是ThreadLocal变量

###### 1.1 不恰当的理解

- ThreadLocal为解决多线程程序的并发问题提供了一种新的思路

- ThreadLocal的目的是为了解决多线程访问资源时的共享问题

- 比较ThreadLocal 与 synchronize 的异同

  

###### 1.2 合理的理解

ThreadLoal 变量，它的基本原理是，同一个 ThreadLocal 所包含的对象（对ThreadLocal< Integer>而言即为 Integer 类型变量），在不同的 Thread 中有不同的副本（实际是不同的实例）。这里有几点需要注意

- 因为每个 Thread 内有自己的实例副本，且该副本只能由当前 Thread 使用。这是也是 ThreadLocal 命名的由来

- 既然每个 Thread 有自己的实例副本，且其它 Thread 不可访问，那就不存在多线程间共享的问题

- 既无共享，何来同步问题，又何来解决同步问题一说？

  

#### 2.ThreadLocal原理

<img src="https://img2018.cnblogs.com/blog/1368768/201906/1368768-20190614000329689-872917045.png" style="zoom:50%;" />

```
ThreadLocal<String> threadLocalA= new ThreadLocal<String>();

线程1： threadLocalA.set("1234");
线程2： threadLocalA.set("5678");
```

<img src="https://mmbiz.qpic.cn/mmbiz_png/KyXfCrME6UKHJicMicOM6durbmt3V1jjmsDUn0adL5zic1XZvEiaVqics9sECEA5CnWAVH9icMVnhK3LPbJ6n3EtI9OA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1" style="zoom:80%;" />

```
ThreadLocal<Integer> threadLocalB = new ThreadLocal<Integer>();

线程1： threadLocalB.set(30);
线程2： threadLocalB.set(40);
```

<img src="https://mmbiz.qpic.cn/mmbiz_png/KyXfCrME6UKHJicMicOM6durbmt3V1jjmsEGbVf8bye3933JYHoGswQdf1FYib0kOUEPzyKYfY9Cdq9n2SxThI4KQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1" style="zoom:80%;" />

#### 3.内存泄漏

实际上 ThreadLocalMap 中使用的 key 为 ThreadLocal 的弱引用，弱引用的特点是，如果这个对象只存在弱引用，那么在下一次垃圾回收的时候必然会被清理掉。

所以如果 ThreadLocal 没有被外部强引用的情况下，在垃圾回收的时候会被清理掉的，这样一来 ThreadLocalMap中使用这个 ThreadLocal 的 key 也会被清理掉。但是，value 是强引用，不会被清理，这样一来就会出现 key 为 null 的 value。

ThreadLocalMap实现中已经考虑了这种情况，在调用 set()、get()、remove() 方法的时候，会清理掉 key 为 null 的记录。如果说会出现内存泄漏，那只有在出现了 key 为 null 的记录后，没有手动调用 remove() 方法，并且之后也不再调用 get()、set()、remove() 方法的情况下。

#### 4.使用场景

###### 4.1 存储用户Session

一个简单的用ThreadLocal来存储Session的例子：

```
private static final ThreadLocal threadSession = new ThreadLocal();

    public static Session getSession() throws InfrastructureException {
        Session s = (Session) threadSession.get();
        try {
            if (s == null) {
                s = getSessionFactory().openSession();
                threadSession.set(s);
            }
        } catch (HibernateException ex) {
            throw new InfrastructureException(ex);
        }
        return s;
    }
```

###### 4.2 解决线程安全的问题

比如Java7中的SimpleDateFormat不是线程安全的，可以用ThreadLocal来解决这个问题：

```
public class DateUtil {
    private static ThreadLocal<SimpleDateFormat> format1 = new ThreadLocal<SimpleDateFormat>() {
        @Override
        protected SimpleDateFormat initialValue() {
            return new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        }
    };

    public static String formatDate(Date date) {
        return format1.get().format(date);
    }
}
```

这里的DateUtil.formatDate()就是线程安全的了。(Java8里的 java.time.format.DateTimeFormatter是线程安全的，Joda time里的DateTimeFormat也是线程安全的）。

