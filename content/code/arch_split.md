---
title: "移动端业务架构"
date: 2018-11-28T18:20:05+08:00
draft: true
---

#### 对架构设计的一些理解：

1. 设计关乎权衡，稍微复杂的系统都有大量的正反馈和负反馈组成。不存在尽善尽美的方案
2. 设计中非常重要的一个工作是：划分角色，确认角色职责，角色间的关系
3. 设计的结果作为对随意的编码行为的限制存在，这些限制代表了设计理念
4. 设计的一个非常重要的目标是管理系统复杂度，通过分层，封装等措施降低整体复杂度，将复杂度进行拆分处理。
5. 设计需要保证系统的整体复杂度主要由系统的本质复杂度组成，限制非本质复杂度的占比；要解决核心问题，并降低次要问题带来的复杂度
6. 没有最好的设计，只有最适合业务场景的设计
7. 系统的复杂度不仅仅来源于业务和技术，还有其它方面，比如：运维需求，数据统计，开发周期，开发效率，工具体系，团队成员的技术背景，财务需求等等等等
8. 如果有一个很烂的系统，划分角色，厘清职责要比处理好业务模块之间的关系更重要一些
9. 移动互联网情景下，移动端的主要复杂度来源于业务的快速变化，业务之间的糅合。
10. 移动端App一般情况下业务链条在比较短，数据一致性需求不高，数据结构距离业务展现近，性能对用户规模不敏感... 这些特性可以大幅简化移动端的架构设计。


## 一. 拆分

#### 1.1 拆分的两种程度：
1. 划分业务边界
2. 编译隔离

#### 1.2 三种拆分工具：
**java package**

优势：java 语言级别支持，使用命名空间进行代码组织，非常灵活。
劣势: 资源没有边界，一个业务模块资源和代码不在一起。同样因为非常灵活，业务边界的维护严重依赖开发同学的自觉性，很难长期保持清晰的业务边界

**gradle source set**

优势：gradle 原生支持，同一个业务模块代码和资源一起管理；代码编译上不隔离

**gradle module**

不同业务间, 代码编译隔离，资源独立；需要的拆分成本非常高

#### 1.3 希望达到的效果
1. 代码和资源都需要按照业务边界来划分和管理
2. 业务模块的划分，可以进行到非常细粒度的层面。非常多的小业务模块，而不是几个大业务模块

```
main
    follow
    rec
    feed
    tabbar
feed
    p_creator
    p_dynamic_detail
    p_widget
setting
    privacy
game
    tab
    video
    subtab
video
    creator
        p_preview
        p_beauity
        p_activities
        p_game
    audience
    multi_link
    link
    pk
    channel
    public_live
    entry
    chatter
    top
user
    login
        p_phone_login
        p_qq_login
        p_wechat_login
        p_weibo_login
    p_bindphone
    p_record
    p_usertag
    p_blacklist
commercial
    p_launcher
imchat

```


> 日常开发中只要有意识的保持业务独立。那么不需要多深的技术能力，多强的设计能力，也不需要花费整块的时间，就可以做到中上水平。是一件付出少，收益大，影响长远的事情。

## 二. 业务模块内架构模式
1. mv* 的各种变种中，V和* 是和业务强绑定的，不应该追求V* 的复用性（下面假设采用MVP模式）
2. 业务模块间有复用性的是M
3. M 包括状态管理和数据类型定义
4. MV* 模式对大一些的项目而言，是业务模块级别的模式，不是App整体级别的设计模式
5. Android 上传播相对广泛，相对成熟的是MVP。 对MVP的理解，请[查看wiki](http://wiki.inkept.cn/pages/viewpage.action?pageId=38546022)

## 三. 模块之间的关系

#### 3.1 最简单的情况，每个业务模块相对独立

![arch_lin_full-context-center.png](https://raw.githubusercontent.com/weixinfree/PickRepo/master/images/arch_lin_full-context-center-dep.png)

1. 每个业务模块内部的设计其实是自由的
2. 推荐业务模块使用MVP模式

---

#### 3.2 业务模块间需要通信，但是通信模式相对简单

![arch_lin_full-context-center.png](https://raw.githubusercontent.com/weixinfree/PickRepo/master/images/arch_lin_full-context-center.png)

通信方式：

1. 启动一个新的页面，或者对话框 （可以通过Route解耦依赖）
2. 从别的业务模块获取数据

注意：

1. M 之间尽量不要相互依赖
2. V, P 可以直接依赖不同业务模块的M，但是一个业务模块的V,P 不能依赖其它模块的V,P
3. 每个业务V,P都可以依赖多个M

---

#### 3.3 业务模块之间需要大量通信，或者深层次的通信，上面的模式不再满足需要

![arch_lin_full-context.png](https://raw.githubusercontent.com/weixinfree/PickRepo/master/images/arch_lin_full-context.png)

1. EventCenter 可以类比为进程内的 MessageQueue；ServiceCenter类比为进程内的RPC。相关的适用范围和使用场景也是类似的
2. 所有的业务模块都可以依赖EventCenter 和 ServiceCenter，但不存在反向依赖
3. EventCenter 和 ServiceCenter 本身没有任何业务，只提供转发

##### EventCenter
1. EventCenter 模式类似App全局的goto，要严格限制使用场景
2. Event通信只适用于业务模块间的通信，限制模块内部使用
3. EventCenter适合于使用 Observer模式的场景

##### ServiceCenter
1. 每个业务模块都可以从ServiceCenter中获取其它模块提供的服务
2. 一个业务模块也可以提供Service给其它模块使用
3. Service接口的设计要遵循最少信息原则，合理封装，不要暴漏太多的实现细节


##### EventCenter和ServiceCenter通信的数据结构定义应该属于谁的职责？
###### 模式1:

1. 将 Event/Struct，Serice接口 放在全局可见的EventCenter 和 ServiceCenter中
2. 优势是相对简单，劣势是通信一多，全局的定义又成了一个非法之地，不能有效的进行管理和拆分

###### 模式2:
1. 将模块的接口层单独抽取出来作为一个独立的module
2. 接口层对于EventCenter是 Event定义
3. 接口层对于ServiceCenter 是 Service接口定义和接口需要的数据结构定义
4. 优势是清晰的依赖，清晰的职责
5. 使用步骤上相对麻烦，对开发同学的自觉性要求也更高一些

推荐优先使用模式1，毕竟绝大部分业务都没有那么复杂

---

#### 3.4 业务模块间关系非常复杂，需要查询/交换大量的状态

![arch_lin_full.png](https://raw.githubusercontent.com/weixinfree/PickRepo/master/images/arch_lin_full.png)

1. 优先考虑使用上面的方式解决，只有整个系统非常复杂，业务模块之间的相互依赖太多，使用统一的状态共享中心明显更有效率的时候，才是合适的使用时机，例如：映客直播间，子业务数量非常大，各个业务模块需要共享很多的状态，使用Context反而会更清晰一点
2. 添加Context角色，Context的角色是上下文信息，维护共享数据
3. Context不一定是单例，但是所有的Context要共享一份实例
4. 应该使用Context，还是直接从其它业务模块的Model查询，还有一个判别标准：当系统需要的非业务数据状态（例如：是否在直播间中，主播liveId，主播uid，是否正在pk，正在连麦，正在播放礼物，是否横屏，软键盘是否弹出）非常多的时候，说明直接查询Model不是特别有效了。这时候使用Context会更合理。非业务数据状态的维护职责不够清晰，反而可以使用Context来解决
5. 从Context查询状态是ok的，Context状态的维护由各个业务共同维护，但是一个状态只能由一个业务模块进行修改，不能多个业务模块同时修改
