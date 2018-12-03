---
title: "内存分析系列之二-你真的理解内存泄漏吗"
date: 2018-11-26T14:01:58+08:00
draft: true
tags: ['memory leak', 'java']
---

首先来看一段Andriod中非常常见的代码，这段代码在AndroidStudio里面会有lint warn，提示有可能有内存泄漏。
今天就来分析下为什么会有内存泄漏，什么情况下会有内存泄漏以及如何解决内存泄漏。

```java
public class MainActivity extends Activity {

    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            ...
        }
    };

    private View demoView;

    @Override
    public void onCreate(Bundle savedInstanceState) {
        setContentView(R.layout.main_activity);
        View view = findViewById(R.id.demo);
        demoView = view;
    }

    @Override
    public void onDestroy() {
        demoView = null;
        mHandler = null;
    }
}
```

### 请回答以下问题
1. 上面的代码有没有内存泄漏？
2. 一定会发生内存泄漏吗？
3. 有了垃圾回收，为什么会发生内存泄漏？怎么定义内存泄漏？
4. 内存泄漏有什么危害？
5. 什么是垃圾？如何判定？算法模型？
6. 对象的引用关系图？
7. demoView = null 有意义吗？
8. mHandler = null 有意义吗？
9. 什么是GCRoot？
10. 怎么解决Handler引起的内存泄漏

---

暂停一会儿，思考一下

---
## 问题解答
解答的关键在于：

1. 能清楚画出对象引用关系图
2. 理解哪些对象是GCRoots

### 对象引用关系图如下
**case 1: onCreate 执行完成，onDestory之前：**

![memory_lean_obj_graph.png](https://raw.githubusercontent.com/weixinfree/PickRepo/master/images/memory_lean_obj_graph.png)

**case 2: onCreate 执行过程中：**

![memory_leak_obj_graph_2.png](https://raw.githubusercontent.com/weixinfree/PickRepo/master/images/memory_leak_obj_graph_2.png)

1. 上面的代码有没有内存泄漏？
如果代码如上面这么简单，那么是没有内存泄漏的。但是mHandler一般

---

## 扩展问题
1. 如何发现内存泄漏？
2. 如何定位内存泄漏？