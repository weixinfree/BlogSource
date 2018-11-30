---
title: "AndPerf Android性能调优工具"
date: 2018-11-29T20:45:12+08:00
draft: false
tags: ['Android', 'performance', '工具效率']
---

这两年做了不少性能优化的工作，总结了一点点性能优化的经验：

1. 性能优化不要相信直觉，不要相信专家，不要相信业务开发者，不要相信自己，不要相信假设，不要相信架构，只相信精确的，合适的性能测量工具
2. make it right before make it faster
3. 不论性能优化，还是设计，认识问题永远是最重要的
4. 实际动手优化前，要先保证指标可测量，可对比
5. 要理解性能数据背后的含义，思维多延伸几步
6. 找到瓶颈点永远是最重要的，如果感觉没有瓶颈点，那么至少80%的可能性不是因为没有瓶颈，而是还没找到
7. 要挑战自己的既有认知，所有不合理的，看起来相互冲突的数据指标，都值得深入挖掘背后的原因
8. 不是所有的细节都能决定成败，但是性能优化是一件 Details matter的事情，深入挖掘，不要浅尝辄止
9. 性能优化是对一个人知识体系的挑战，是技术上融会贯通的开始
10. 对于编程来说，太阳底下无新事。你遇到的问题，肯定有人已经遇到过了，而且多半还有好用的性能工具提供出来。
11. 从底层领域寻找解决方案往往是突破口，关于操作系统机制的理解非常有用；学习和使用Linux性能测试工具集也是一个好的开始
12. 性能测试工具有自身的局限，要了解性能测试工具的机制
13. 只相信测量结果，但不要完全相信测量结果，多测试几遍
14. 磨刀不误砍柴工，找到合适的工具并使用好，就成功了一小半
15. 做什么事情，都不要一上来就埋头苦干。骄傲的程序员要追求十倍效率
16. 要提升效率，一定要打造自己的工具集

如果你认同上面的观点，那么工具的重要性不言而喻。最近把工作中自己写的一些工具统一的整理了一下，组装了一个工具类，作为Linux和Android系统现有工具集的一个补充和扩展，发布到了[pypi](https://pypi.org/project/andperf/)。欢迎各位小伙伴试用。

## 安装
```bash
pip3 install andperf
```

## 使用

#### andperf dev-screen
![dev_screen.png](https://raw.githubusercontent.com/weixinfree/AndPerf/master/images/dev_screen.png)

#### andperf stat-thread
![stat_t.png](https://raw.githubusercontent.com/weixinfree/AndPerf/master/images/stat_t.png)

#### andperf top-activity
![top_activity.png](https://raw.githubusercontent.com/weixinfree/AndPerf/master/images/top_activity.png)

#### andperf fps
![fps.png](https://raw.githubusercontent.com/weixinfree/AndPerf/master/images/fps.png)

#### andperf gfx-hist
![gfx_historgram.png](https://raw.githubusercontent.com/weixinfree/AndPerf/master/images/gfx_historgram.png)

#### andperf meminfo-pie
![meminfo_pie.png](https://raw.githubusercontent.com/weixinfree/AndPerf/master/images/meminfo_pie.png)

#### andperf meminfo-trend
![meminfo_trend.png](https://raw.githubusercontent.com/weixinfree/AndPerf/master/images/meminfo_trend.png)

## 完整命令列表

```bash
andperf config                  设置用户自定义配置
andperf cpuinfo                 查看
andperf dev-mem                 查看设备内存信息
andperf dev-screen              查看设备屏幕信息
andperf dump-config             查看当前的用户自定义配置
andperf dump-layout             导出当前栈顶Activity布局，并在浏览器打开
andperf fps                     计算fps，最后会绘制一张fps变化图
andperf gfx-hist                查看gfx每帧绘制耗时分布直方图
andperf gfx-reset               reset app 的gfxinfo，重新开始统计
andperf gfxinfo                 查看app的gfxinfo信息
andperf meminfo                 查看app的meminfo信息
andperf meminfo-pie             将当前app的各部分内存占用按照饼图展示
andperf meminfo-trend           展示app各部分内存随时间的变化
andperf screencap               截图并在浏览器打开
andperf stat-thread             统计一段时间内app进程内，各线程获得到的时间片占比
andperf systrace                调用Android systrace 命令，并在chrome中打开
andperf top-activity            查看当前栈顶Activity
andperf top-app                 查看当前栈顶App
```

## config
config 指定app package name，可以在执行其它指令时节省很多输入

```bash
andperf config --app=com.meelive.ingkee
```