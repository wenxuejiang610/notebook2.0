title: Java如何执行一个最简单的程序
tag: JVM
---

本篇为学习JAVA虚拟机的第一篇文章，需要之前对JVM有一定了解的基础。我们都知道，JAVA号称：一次编译多处运行。这就离不开字节码文件和虚拟机啦！那么，虚拟机到底是如何去执行一个简单的程序的呢？理解了这个，我们就可以理解java时如何做到平台无关的了。下面我们来分析分析。
<!-- more -->

首先，写一个最简单的程序：


```java
public class Main {
    public static void main(String[] args) {
        int i=1,j=5;
        i++;
        ++j;
        System.out.println(i);
        System.out.println(j);
    }
}
```

运行之后的结果想必就一目了然，我们就通过这个程序来分析分析到底是怎么执行这个程序额的。

首先呢，java程序的执行经历编译，编译成系统能识别的文件，这里的系统对应java语言就是JVM，即JAVA虚拟机。JVM在识别之后，再去与我们真正的操作系统进行交互和处理。

所以，我们要执行一个.java程序，必须要先进行编译。初学者都会学习一个指令叫做`javac`：

![image](http://bloghello.oursnail.cn/javabasic6-1.png)

我们会发现路径下面就会多一个.class文件，这就是编译之后的文件。直接点开：


```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

public class Main {
    public Main() {
    }

    public static void main(String[] var0) {
        byte var1 = 1;
        byte var2 = 5;
        int var3 = var1 + 1;
        int var4 = var2 + 1;
        System.out.println(var3);
        System.out.println(var4);
    }
}
```
我们看第一行注释，说的是编译后的文件已经自动被`IDEA`反编译了，所以我们还能看得懂。真正的文件是：


```
漱壕   4 
  	  
     <init> ()V Code LineNumberTable main ([Ljava/lang/String;)V 
SourceFile 	Main.java         Main java/lang/Object java/lang/System out Ljava/io/PrintStream; java/io/PrintStream println (I)V !                    *? ?    	        	 
      E     <=??? ? ? ? ? 
```


我们可以看到，其实是一堆乱码，根本看不懂。而在执行的时候，class文件是一种8位字节的二进制流文件。放在`sublime`中可以看到二进制文件（以16进制显示，在JAVA虚拟机中将来了解这各文件的含义，我们可以看到第一个单词是cafe babe，表明这是一个class字节码文件）：

![image](http://bloghello.oursnail.cn/javabasic6-2.png)

那么我们想看看.class中的信息，还是需要反编译，这个时候可以用`javap`指令来做。如果我们对其不熟悉，可以先执行`javap -help`来了解了解。



```
用法: javap <options> <classes>
其中, 可能的选项包括:
  -help  --help  -?        输出此用法消息
  -version                 版本信息
  -v  -verbose             输出附加信息
  -l                       输出行号和本地变量表
  -public                  仅显示公共类和成员
  -protected               显示受保护的/公共类和成员
  -package                 显示程序包/受保护的/公共类
                           和成员 (默认)
  -p  -private             显示所有类和成员
  -c                       对代码进行反汇编
  -s                       输出内部类型签名
  -sysinfo                 显示正在处理的类的
                           系统信息 (路径, 大小, 日期, MD5 散列)
  -constants               显示最终常量
  -classpath <path>        指定查找用户类文件的位置
  -cp <path>               指定查找用户类文件的位置
  -bootclasspath <path>    覆盖引导类文件的位置
```

我们注意到，有一个`-c`是进行反汇编，那么就用它试试:


```
E:\JavaBasic\src>javap -c Main.class


Compiled from "Main.java"
public class Main {
  public Main();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: iconst_1
       1: istore_1
       2: iconst_5
       3: istore_2
       4: iinc          1, 1
       7: iinc          2, 1
      10: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
      13: iload_1
      14: invokevirtual #3                  // Method java/io/PrintStream.println:(I)V
      17: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
      20: iload_2
      21: invokevirtual #3                  // Method java/io/PrintStream.println:(I)V
      24: return
}
```

那么这反汇编出来的东西是什么呢？这是一连串的指令，其实这些是加载class文件时真正执行的java虚拟机指令。


我们来看看它的含义吧！

![image](http://bloghello.oursnail.cn/javabasic6-3.png)


仔细看看，其实发现并不神秘，一个函数的执行是一个入栈出栈的过程。ok，大体了解了字节码文件是什么以及里面的指令含义之后，我们对java如何执行它已经大体清楚了。下面执行一下：

那么如何运行呢？

![image](http://bloghello.oursnail.cn/javabasic6-4.png)

其实这是废话，初学java其实是`java Main`运行的：

```
E:\JavaBasic\src>java Main

2
6
```

这个时候，class文件可以移植到任何平台上去，比如直接上传到`linux`上，只要JDK或者JRE环境类似即可，就可以直接运行了，不需要编译，也不需要关心是什么系统。这就做到了一次编译到处运行。

下面总结一下：

![image](http://bloghello.oursnail.cn/javabasic6-5.png)

Java源码首先被编译成字节码，再由不同平台的JVM进行解析，JAVA语言在不同平台上运行时不需要进行重新编译，JAVA虚拟机在执行字节码的时候，把字节码转换为具体平台上的机器指令，然后各种操作系统就可以正确识别了。这就是JAVA如何执行代码和平台无关性的原因。