---
title: "内存分析系列之一-这么简单的代码你真的理解了吗?"
date: 2018-11-26T14:01:55+08:00
draft: false
tags: ['java', '函数调用栈', 'JVM', 'Android', '基础', '线程']
---

# Details makes perfect

负责技术面试挺长时间了，发现很多社招/校招同学对一些非常基础的问题有着“惊人”的掌握程度。和团队内的小伙伴聊了一下，发现也不是很清晰，因此有了这一系列的关于内存的分享和讨论。形式上采用了代码实例+问题引导的方式，推荐看文章的小伙伴先不要看答案，自己回答一遍，这些看起来非常简单的代码真的理解了吗？

## Talk is cheap, Show me the code

```java
class Person {
  int age;
  String name;
}

public void demo() {
  Person xm = new Person();
  xm.name = "小明";
  xm.age += 10;
}
```

代码非常简单，核心代码只有三行，请回答以下几个问题:

1. 一个线程执行demo方法，内存分配情况
2. 堆和栈的内存布局
3. 内存分配大小
4. demo 方法是线程安全的吗？
5. 三个线程执行demo方法，内存分配情况
6. demo方法执行完成之后，那些内存可以回收？回收时机？

暂 停 一 小 会 儿 , 思 考 一 下

---

## 问题解答
1) 一个线程执行demo方法，内存分配情况？

2）堆和栈的内存布局

3）内存分配大小

![memory_layout](https://raw.githubusercontent.com/weixinfree/PickRepo/master/images/memory_layout.png)

##### 值得注意的有以下几点:

1. int 是值类型，所以xm.age 存储的是直接值10
2. String 是引用类型，所以xm.age是一个指向常量池（或者其它对象）的引用
3. 成员变量也是分配在堆上的
4. 一个对象的实际大小和虚拟机实现相关，例如需要额外的空间存储指向class的指针，存储gc分代年龄等，为了效率考虑，可能还会有字节对齐填充
5. 栈上会有一个局部变量数组，存储方法参数和局部变量
6. 引用的大小，一般在32位机器上是32位/4字节，64位机器上为64位/8字节
7. 执行demo方法的时候，函数调用栈需要压入一个函数栈帧。栈帧负责代码的执行过程中中间变量的存储等，还有更精细的结构，这个后面会讲到
8. 堆上的对象各个成员变量之间一般会是连续的
9. 堆提供了一个连续的大块存储空间抽象，实际的物理内存因为虚拟内存分页的存在，物理上不是连续的（每页一般是4K）

##### 栈帧的细致结构
![frame](https://raw.githubusercontent.com/weixinfree/PickRepo/master/images/frame.png)

##### JVM 规范对Frame的介绍
> 2.6 Frames

> A frame is used to store data and partial results, as well as to perform dynamic
linking, return values for methods, and dispatch exceptions.

> A new frame is created each time a method is invoked. A frame is destroyed when
its method invocation completes, whether that completion is normal or abrupt (it
throws an uncaught exception). Frames are allocated from the Java Virtual Machine
stack (§2.5.2) of the thread creating the frame. Each frame has its own array of
local variables (§2.6.1), its own operand stack (§2.6.2), and a reference to the runtime
constant pool (§2.5.5) of the class of the current method.

大概意思如下:

1. 每个frame在方法调用的时候创建，在方法结束的时候销毁
2. frame由java线程stack管理
3. 每个frame至少包含三部分:
  - 局部变量数组 （局部变量编译后会是数组的形式）
  - 操作数栈
  - 到当前类的常量池引用
4. 如果是成员方法，局部变量数组第一个位置是this local var[0]
5. 如果方法有n个参数，参数占据local var[1...n+1]
6. 方法中定义的局部变量在 local var[n+2...]

#### demo方法的内存足迹是： 

假设不考虑常量池，方法区和各个不同虚拟机的实现细节; 假设机器字长为64位

1. 栈帧中局部变量数组大小为
  - this(ref) -> 8字节
  - 方法参数（0）-> 0 字节
  - xm(ref) -> 8字节 
2. 栈帧中操作数栈:
[JVM 指令](https://docs.oracle.com/javase/specs/jvms/se10/html/jvms-6.html#jvms-6.5.athrow)
(字节码见下一小节)
  - 操作数栈最大深度大概为3-4， -> 30字节左右（ref是8字节，int是2字节）
3. 栈帧中常量池指针（ref）-> 8字节
3. 堆上分配的对象
  - xm.age (int) -> 4字节
  - xm.name (string/ref) -> 8字节
  - 对象头，对齐等（暂不考虑）-> 0字节(大约在十几到几十字节)

最小内存足迹大概是：
stack(16 + 30 + 8 = 54) + heap(4 + 8 = 12) = 66字节

考虑到对象头，字节对齐，debug信息等等实现细节，量级应该在100字节左右。
另一个值得注意的一点是，这种小方法，函数栈帧内存足迹远大于heap内存足迹

#### 4）demo 方法是线程安全的吗？

很多同学会立马回答: 肯定是非线程安全的，因为`xm.age += 10`不具有原子性。很不幸，这个回答其实是错的，并且至少犯了两个错误，请思考下面这两个问题：

1. 没有原子性的操作一定是非线程安全的吗？
2. 一个方法内所有的操作都是原子的，这个方法就是线程安全的吗？

上面的两个问题的正确答案都是否。

单线程是没有线程安全问题的，不管编译器/虚拟机优化，各种重排序，cpu乱序/流水线执行，高速缓存cache等等措施，都有一个非常基本的限制条件：as-if-serial/intra sematics 语义: 无论怎么优化，都不能改变代码的单线程执行语义，要不然就没有可能好好的进行玩耍了。

这就引出了另一个问题，intra sematics 语义并没有规定多线程执行时的语义，现实世界中各个cpu厂商也定义了不同的缓存一致性协议，所以，其实操作系统和JVM并没有为程序员屏蔽掉硬件层的全部复杂度。不过好在，JVM定义了统一的规范，来屏蔽各个系统，cpu的缓存一致性差异。毕竟对于计算机系统而言，没有什么是一层抽象解决不了的，如果有，那就再加一层抽象。

出现线程安全问题的前提条件是有‘竞态条件’。在这段代码中并没有全局变量，所有的对象都是局部变量，没有竞态条件，也就自然没有线程安全问题。所以非原子性操作，只要是线程封闭的，也是线程安全的

同样的，只要有竞态条件，即使每一个操作都是原子的，也不能保证复合操作是原子性的，因此没有正确同步的方法，依然存在线程安全问题。

回到当前的问题: **demo方式是线程安全的，因为没有静态变量。**

不过可以继续思考一个有意思的问题，`xm.age += 10`不是原子性的，那么具体是几条指令呢？

```bash
public class Demo {
  public Demo();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public void demo();
    Code:
       0: new           #2                  // class Demo$Person
       3: dup
       4: invokespecial #3                  // Method Demo$Person."<init>":()V
       7: astore_1
       8: aload_1
       9: ldc           #4                  // String 小明
      11: putfield      #5                  // Field Demo$Person.name:Ljava/lang/String;
      14: aload_1
      15: dup
      16: getfield      #6                  // Field Demo$Person.age:I
      19: bipush        10
      21: iadd
      22: putfield      #6                  // Field Demo$Person.age:I
      25: return
}
```

如果不熟悉java字节码，可以阅读[基于栈的虚拟机](https://weixinfree.github.io/blog/code/calc/)

可以看到语句：`xm.age += 10`至少可以拆解为下面6条指令，确实没有原子性。但这依然不妨碍这段代码的线程安全性

```bash
14: aload_1
15: dup
16: getfield      #6                  // Field Demo$Person.age:I
19: bipush        10
21: iadd
22: putfield      #6                  // Field Demo$Person.age:I
```

同样的，`xm.name = "小明";` 也是多条指令
```bash
 8: aload_1
 9: ldc           #4                  // String 小明
11: putfield      #5                  // Field Demo$Person.name:Ljava/lang/String;
```
> 注意：不要错误的理解为每条jvm虚拟机指令都具有原子性的。
> 原子性，可见性，有序性相关的问题，可以阅读Java语言规范，Java内存模型，JSR131，JST133等资料

#### 5）三个线程执行demo方法，内存分配情况

三个线程执行和一个线程执行的情况类似，不同之处在于：

1. 每个线程有自己的函数调用栈，所以是三个独立的栈，三个新压栈的栈帧。
2. 堆是线程共享的，只有一个
3. 堆上会分配三个Person对象
4. “小明”这个字符串只在常量池存在一份
5. 会存在三个10，堆上的三个Person对象，每个都有一个age field，值都是10。因为int是值类型
6. 没有线程安全问题，不会多分配对象，也不会少分配
7. 总的内存足迹是 (stack(16 + 30 + 8 = 54) + heap(4 + 8 = 12) = 66) * 3，大概是300字节量级

#### 6）demo方法执行完成之后，那些内存可以回收？回收时机？

1. 方法执行完成之后, 方法栈帧就可以销毁了。这个可以是立即销毁的，也可以由虚拟机实现决定，JVM标准并没有限制
2. 堆上的Person对象因为没有了引用，也可以进行回收，但是因为GC是不确定的，所以回收时间是不确定的

如果感觉自己回答的不是很好，推荐你看另一篇文章: **[如何建立一个具体的，顺畅的，细节丰富的知识体系模型？](blog/holmes/feymn_model.md)**

---

## 扩展问题一
如果你已经理解了上面的问题, 请尝试挑战下这个稍微更改过的，依然十分简单的代码：
```java
class Student extends Person {
  Person classmate;
}

public void demo() {
  Student xm = new Student();
  xm.name = "小明";
  xm.age += 10;

  Person xh = new Student();
  xh.name = "小红";
  xh.age = 12;

  xm.classmate = xh;
}
```

**问题和之前保持一致：**

1. 一个线程执行demo方法，内存分配情况
2. 堆和栈的内存布局
3. 内存分配大小
4. demo 方法是线程安全的吗？
5. 三个线程执行demo方法，内存分配情况
6. demo方法执行完成之后，那些内存可以回收？回收时机

---

## 扩展问题二

1. java 线程和操作系统线程的关系
1. 创建一个线程的开销
2. 线程的栈有多大？
3. 线程切换成本？为什么线程要比进程轻量级？
4. Android中main线程和其它线程的最大大小？
5. 线程栈造成的OOM ? 什么情况下会出现?


## 参考资料
[JVM 虚拟机规范](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-2.html)
