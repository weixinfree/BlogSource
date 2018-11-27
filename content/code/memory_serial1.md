---
title: "内存分析系列之一-这么简单的代码你真的理解了吗?"
date: 2018-11-26T14:01:55+08:00
draft: false
---

负责技术面试挺长时间了，发现很多开发同学对一些非常基础的问题有着惊人的掌握程度😱（当然是非常差的那一端）。团队内部的小伙伴也不是很清晰，因此有了这一系列的分享和讨论。

编写单文件/class级别的代码，非常能够体现一个工程师水平的有以下几点：

1. 线程管理能力
2. 方法管理能力（这个在[如何编写简单清晰的代码]()中介绍过了）
3. 状态管理能力
4. null的管理能力 （状态管理的一个特例）

本系列讨论的就是关于状态管理非常非常重要的基础-对内存的理解。形式上采用了代码实例+问题引导的方式，看文章的小伙伴可以先不看答案，自己回答一遍，看下自己的理解程度。

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

代码真的非常简单，简单到会有很多人不由自主的嗤之以鼻。但是先别着急看答案，也别着急对这个简单的问题嗤之以鼻，认真做做看，然后对比下结果，依据拍脑袋，不靠谱统计结果有90%以上的可能性，你可以在接下来的讨论中可以发现自己的不足，学到新的东西。

**做好挑战的准备了吗？**

**问题：**

1. 一个线程执行demo方法，内存分配情况
2. 堆和栈的内存布局
3. 内存分配大小
4. demo 方法是线程安全的吗？
5. 三个线程执行demo方法，内存分配情况
6. demo方法执行完成之后，那些内存可以回收？回收时机

## 问题解答
### 1) 一个线程执行demo方法，内存分配情况？
### 2）堆和栈的内存布局
![hh](https://raw.githubusercontent.com/weixinfree/PickRepo/master/images/memory_layout.png)

值得注意的有以下几点:

1. int 是值类型，所以xm.age 存储的是直接值10
2. String 是引用类型，所以xm.age是一个指向常量池（或者其它对象）的引用
3. 成员变量也是分配在堆上的（并没有一个单独的存储变量的栈）
4. 一个对象的实际大小和实现相关，例如可能需要额外的信息指向class，存储gc分代年龄等，为了效率考虑，可能还会有字节对齐
5. 栈上会有一个局部变量数组，存储方法参数和局部变量
6. 引用的大小是和实现相关的，一般在32位机器上是32位/4字节，64位机器上为64位/8字节
7. 执行demo方法的时候，函数调用栈需要压入一个函数栈帧。函数调用栈负责代码的执行过程中中间变量的存储等，有更精细的结构，例如：堆栈虚拟机依赖的操作数栈


#### 栈帧的细致结构
![frame](https://raw.githubusercontent.com/weixinfree/PickRepo/master/images/frame.png)


### 3）内存分配大小

### 4）demo 方法是线程安全的吗？

虽然 `xm.age += 10;` 是非原子性的，可以拆分成多个不同的指令
但是这个方法确实是线程安全的，因为**没有竞态条件**。

这就引出了一个常见的误解：不具备原子性的操作，一定是线程不安全的吗？
```python
while (没想通) {
    // 卡住是完善知识体系的开始
    think and analysis; 
}
```

上面代码的字节码如下:

```java
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

如果不熟悉java字节码，可以阅读[基于栈的虚拟机](code/calc.md)

可以看到语句：`xm.age += 10`至少可以拆解为下面6条指令，确实没有原子性。但这依然不妨碍这段代码的线程安全性

```
14: aload_1
15: dup
16: getfield      #6                  // Field Demo$Person.age:I
19: bipush        10
21: iadd
22: putfield      #6                  // Field Demo$Person.age:I
```


### 5）三个线程执行demo方法，内存分配情况
### 6）demo方法执行完成之后，那些内存可以回收？回收时机？

如果你达到了上面的理解/了解程度，那么你的java基础能力应该还不错。很多来我司面试架构师，技术专家职位的人，我都会问他这个问题，如果回答的支支吾吾，或者有明显错误，逻辑冲突，就会非常影响对候选人的技术评判。天天做系统架构，性能优化，如果连这么简单/基础的代码都理解/分析不了，那么怎么保证不是瞎优化呢。

轻松通过的小伙伴，请自动略过下面这一小节，另外麻烦简历请发送至[email](wangw@inke.cn)!!!

## 如果你受到了打击🔨

根据在公司内的讨论情况，大部分人可以归结为以下几种情况：

1. 关键知识点缺失，无法回答上来
2. 几乎每个知识点都了解，但是没有串起来，而且回答问题的时候发现不同的知识点相互矛盾
3. 大概都知道，也差不多，但是细节不清晰

如果你的情况也类似，那么刚好我有药😜。治病药方是：

: 你需要一个具体的，顺畅的，细节丰富的关于程序运行的模型.

*如何建立一个具体的，顺畅的，细节丰富的知识体系模型呢？请查看[构建知识体系的费曼方法](blog/holmes/feymn_model.md)*



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

## 扩展问题二

1. java 线程和操作系统线程的关系
1. 创建一个线程的开销
2. 线程的栈有多大？
3. 线程切换成本？为什么线程要比进程轻量级？
4. Android中main线程和其它线程的最大大小？
5. 线程栈造成的OOM ? 什么情况下会出现?


## 参考资料
[JVM 虚拟机规范](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-2.html)
