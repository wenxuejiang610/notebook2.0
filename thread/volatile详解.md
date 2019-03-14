title: volatile详解
tag: java多线程
---
volatile是比较重要的关键字，它涉及JMM，我们需要对其进行深入了解。
<!-- more -->

## 一、java内存模型JMM

JMM本身是一种抽象的概念，并不真实存在，它描述的是一组规则或规范，通过这组规范定义了程序中各个变量(包括实例字段，静态字段和构成数组对象的元素)的访问方式。

请务必区分HMM和JAVA内存区域，JMM描述的是一组规则，围绕原子性、有序性以及可见性展开。

![image](http://bloghello.oursnail.cn/thread8-1.png)

大多数的变量是只能存储在主内存中的，线程也不能直接去主内存中读取数据，而是获取数据的副本，每个线程对这个副本进行修改后，会在某个时机刷新回主内存。每个线程之间的工作内存的值是互不透明的，因此不能互相访问，线程间的通信必须通过主内存来完成。

## 二、JMM主内存和工作内存都放些什么

- 主内存
    - 存储JAVA实例对象
    - 包括实例变量、类信息、常量、静态变量等
    - 属于数据共享的区域，多线程并发操作时会引起线程安全问题
- 工作内存
    - 存储当前方法的所有本地变量信息，本地变量对其他线程不可见(方法里的基本数据类型会直接被存储在工作内存的栈帧结构中)
    - 字节码行号指示器、Native方法信息
    - 如果是引用类型，引用存储在工作内存中，实例存储在主内存中
    - 属于线程私有数据区域，不存在线程安全问题
    
![image](http://bloghello.oursnail.cn/thread8-2.jpg)

## 三、指令重排序

为了提高执行性能，JVM会进行一定的指令重排序，禁止方式就是加入内存屏障指令，下面会说。

当然了，指令重排序需要满足一定的条件：

- 在单线程环境下不能改变程序运行的结果
- 存在数据依赖关系的不允许重排序

无法通过`happend-before`原则推导出来的，才能进行指令的重排序。

## 四、happend-before

多线程有两个基本的问题，就是原子性和可见性，而`happens-before`规则就是用来解决可见性的。

即：在时间上，动作A发生在动作B之前，能不能**保证**B可以看见A？如果可以保证的话，那么就可以说`hb(A,B)`


```java
class VolatileExample {
    int a = 0;
    volatile boolean flag = false;

    public void writer() {
        a = 1;                   //1
        flag = true;               //2
    }

    public void reader() {
        if (flag) {                //3
            int i =  a;           //4
            ……
        }
    }
}
```

假设线程A执行`writer()`方法之后，线程B执行`reader()`方法。根据happens before规则，这个过程建立的happens before 关系可以分为两类：

* 根据程序次序规则，1 happens before 2; 3 happens before 4。
* 根据volatile规则，2 happens before 3。
* 根据happens before 的传递性规则，1 happens before 4。


上述`happens before` 关系的图形化表现形式如下：

![image](http://bloghello.oursnail.cn/thread8-3.png)

在上图中，每一个箭头链接的两个节点，代表了一个`happens before` 关系。黑色箭头表示程序顺序规则；橙色箭头表示`volatile`规则；蓝色箭头表示组合这些规则后提供的`happens before`保证。

这里A线程写一个`volatile`变量后，B线程读同一个`volatile`变量。A线程在写`volatile`变量之前所有可见的共享变量，在B线程读同一个`volatile`变量后，将立即变得对B线程可见。

说了那么多，java中是如何保证这种可见性的呢？`Volatile`闪亮登场。



## 五、什么是volatile

`volatile`关键字的目的是保证被它修饰的共享变量对所有线程总是可见的。


## 六、为什么要用volatile

`Volatile`变量修饰符如果使用恰当的话，它比`synchronized`的使用和执行成本会更低，因为它不会引起线程上下文的切换和调度。

一旦一个共享变量（类的成员变量、类的静态成员变量）被`volatile`修饰之后，那么就具备了两层语义：

- 保证了不同线程对这个变量进行操作时的可见性，即一个线程修改了某个变量的值，这新值对其他线程来说是立即可见的。
- 禁止进行指令重排序。

## 七、volatile如何保证可见性

`voliatile`关键字保证了在进程中变量的变化的可见性。

在多线程的应用里，如果线程操作了一个没有被`volatile`关键字标记的变量，那么每个线程都会在使用到这个变量时从主存里拷贝这个变量到CPU的`cache`里面（为了性能！CPU缓存可比内存快多了）。如果你的电脑有多于一个CPU，那么每个线程都会在不同的CPU上面运行，这意味着每个线程都会把这个变量拷贝到不同的CPU `cache`里面，正如下图所示：

![image](http://bloghello.oursnail.cn/thread8-4.png)

一个不带有`volatile`关键字的变量在JVM从主存里面读取数据到CPU cache或者从cache里面写入数据到主存时是没有保证的。


想象这样一个场景，当一到两个线程允许去共享一个包含了一个计数变量的对象，这个计数变量如下所定义


```java
public class SharedObject {

    public int counter = 0; //无关键字

}
```

然后，这线程一增加了`counter`变量的值，但是，但是同时线程一和线程二都有可能随时读取这个`counter`变量。

如果这个`counter`变量未曾使用`volatile`声明，那么我们就无法保证这个变量在两个线程中所位于的CPU的cache和主存中的值是否保持一致了。示意图如下： 

![image](http://bloghello.oursnail.cn/thread8-5.png)


那么部分的线程就不能看到这个变量最新的样子，因为这个变量还没有被线程写回到主存中，这就是可见性的问题，这个线程更新的变量对于其他线程是不可视的。

在声明了`counter`变量的`volatile`关键字后，所有写入到`counter`变量的值会被立即写回到主存中。同时，所有读取这个变量的线程会先把对应的工作内存置为无效，从主存里面读取这个变量，下面的代码就是声明带`volatile`关键字的变量的方法


```java
public class SharedObject {

    public volatile int counter = 0;

}
```
如此声明这个变量就保证了这个变量对于其他写这个变量的线程的可见性。

总结：

处理器为了提高处理速度，不直接和内存进行通讯，而是先将系统内存的数据读到内部缓存（L1,L2或其他）后再进行操作，但操作完之后不知道何时会写到内存，如果对声明了`Volatile`变量进行写操作，JVM就会向处理器发送一条Lock前缀的指令，将这个变量所在缓存行的数据写回到系统内存。但是就算写回到内存，如果其他处理器缓存的值还是旧的，再执行计算操作就会有问题，所以在多处理器下，为了保证各个处理器的缓存是一致的，就会实现缓存一致性协议，每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期了，当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置成无效状态，当处理器要对这个数据进行修改操作的时候，会强制重新从系统内存里把数据读到处理器缓存里。



## 八、来详细说说volatile写-读的内存语义

**volatile写的内存语义如下**：

> 当写一个`volatile`变量时，JMM会把该线程对应的本地内存中的共享变量刷新到主内存。

以上面示例程序`VolatileExample`为例，假设线程A首先执行`writer()`方法，随后线程B执行`reader()`方法，初始时两个线程的本地内存中的flag和a都是初始状态。下图是线程A执行`volatile`写后，共享变量的状态示意图：

![image](http://bloghello.oursnail.cn/thread8-6.png)

如上图所示，线程A在写flag变量后，本地内存A中被线程A更新过的两个共享变量的值被刷新到主内存中。此时，本地内存A和主内存中的共享变量的值是一致的。

**volatile读的内存语义如下**：

> 当读一个`volatile`变量时，JMM会把该线程对应的本地内存置为无效。线程接下来将从主内存中读取共享变量。

![image](http://bloghello.oursnail.cn/thread8-7.png)

如上图所示，在读flag变量后，本地内存B已经被置为无效。此时，线程B必须从主内存中读取共享变量。线程B的读取操作将导致本地内存B与主内存中的共享变量的值也变成一致的了。

如果我们把`volatile`写和`volatile`读这两个步骤综合起来看的话，在读线程B读一个`volatile`变量后，写线程A在写这个`volatile`变量之前所有可见的共享变量的值都将立即变得对读线程B可见。

下面对`volatile`写和`volatile`读的内存语义做个总结：

* 线程A写一个`volatile`变量，实质上是线程A向接下来将要读这个`volatile`变量的某个线程发出了（其对共享变量所在修改的）消息。
* 线程B读一个`volatile`变量，实质上是线程B接收了之前某个线程发出的（在写这个`volatile`变量之前对共享变量所做修改的）消息。
* 线程A写一个`volatile`变量，随后线程B读这个`volatile`变量，这个过程实质上是线程A通过主内存向线程B发送消息。

## 九、volatile如何禁止指令重排序

这就不得不提一个指令叫做：内存屏障了。

它可就厉害了，

- 保证特定操作的执行顺序
- 保证某些变量的内存可见性

通过插入内存屏障指令禁止在内存屏障前后的指令执行重排序优化。

这个指令对编译器和CPU的执行都是起作用的，可用强制刷出各种CPU的缓存数据，因此任何CPU上的线程都能读取到这些数据的最新版本。

因此，从根本上来说，是内存屏障指令实现了`volatile`的可见性和禁止指令重排序的。

## 十、volatile的应用场景

> `volatile`关键字只能对32位和64位的变量使用

`synchronized`关键字是防止多个线程同时执行一段代码，那么就会很影响程序执行效率，而`volatile`关键字在某些情况下性能要优于`synchronized`，但是要注意`volatile`关键字是无法替代`synchronized`关键字的，因为`volatile`关键字无法保证操作的原子性。通常来说，使用`volatile`必须具备以下2个条件：

> 1）对变量的写操作不依赖于当前值

> 2）该变量没有包含在具有其他变量的不变式中

下面列举几个Java中使用`volatile`的几个场景。

①.状态标记量
```java
volatile boolean flag = false;
 //线程1
while(!flag){
    doSomething();
}
  //线程2
public void setFlag() {
    flag = true;
}
```


②.单例模式中的`double check`
```java
class Singleton{
    private volatile static Singleton instance = null;
 
    private Singleton() {
 
    }
 
    public static Singleton getInstance() {
        if(instance==null) {
            synchronized (Singleton.class) {
                if(instance==null)
                    instance = new Singleton();//非原子操作
            }
        }
        return instance;
    }
}
```

> `instance = new Singleton();`//非原子操作

执行这一句，JVM发生了如下事情：

- 给 `instance` 分配内存
- 调用 `Singleton` 的构造函数来初始化成员变量
- 将`instance`对象指向分配的内存空间（执行完这步 `instance` 就为非 `null` 了）

但是在 JVM 的即时编译器中存在指令重排序的优化。也就是说上面的第二步和第三步的顺序是不能保证的，最终的执行顺序可能是 1-2-3 也可能是 1-3-2。如果是后者，则在 3 执行完毕、2 未执行之前，被线程二抢占了，这时 `instance` 已经是非 `null` 了（但却没有初始化），所以线程二会直接返回 `instance`，然后使用，然后顺理成章地出错了，不再是单例了。
