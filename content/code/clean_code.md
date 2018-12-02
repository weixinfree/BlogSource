---
title: "编写清晰代码的几个简单技巧"
date: 2018-11-22T13:45:09+08:00
draft: true
---

为什么要编写清晰的代码？

1. 一个软件系统的维护周期远远长于开发周期；
2. 代码是人来写的，人来读的，人来修改维护的，只是恰好可以让机器执行。现在硬件性能非常高的情况下，人才是开发和维护软件系统的瓶颈

## 编写清晰代码基本原则
1. 可见性最小
2. 变化最少
3. 距离上下文最近

这几个原则非常

### 1. 最小缩进原则

**例子:**

Negative:
```java
if (user != null) {
    if (user.name != null && user.age > 10) {
        // do something
    }
}
```
Positive:
```java
if (user == null) return;
if (user.name == null) return;
if (user.age <= 10) return;

// do something
```
Positive 2:
```java
do {
    if (user == null) break;
    if (user.name == null) break;
    if (user.age <= 10) break;

    // do something
} while(false)
```

**思考: 是什么导致了缩进？**

1. 本质上是业务逻辑需求，需要有分支，循环等逻辑，这就要求使用`if else, for , while`，从而造成缩进
2. 大量的分支逻辑，其实是卫语句

> 最小化缩进 => 减少卫语句对主干逻辑的影响

### 2. 作用域最小化

### 3. 单一抽象层次

### 4. 最简单变化

### 5. 复合表达式命名
