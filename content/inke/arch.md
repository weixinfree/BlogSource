---
title: "代码质量和开发效率提升构思"
date: 2018-11-28T18:20:05+08:00
draft: true
---

## 理想情况？？
1. 良好的业务架构设计，可控的历史债务
2. 完善的工具支持
3. 流畅的工作flow
4. 优秀的开发人员

## 初期目标

- 梳理业务层面的整体架构设计；
- 逐步偿还历史债务；
- 合适的git flow，保证架构设计落地

**怎么到达目标？**

1. 业务层按照业务边界拆分和维护
2. 单业务模块层面设计模式有标准参考设计
3. 业务模块间关系和大型业务模块怎么处理
4. 分支模型，PR机制和代码review

## 一. 拆分

**1.1 拆分的两种程度：**

1. 划分业务边界
2. 编译隔离

**1.2 三种拆分工具：**

java package:

- 优势：java 语言级别支持，使用命名空间进行代码组织，非常灵活。
- 劣势：资源没有边界，一个业务模块资源和代码不在一起。同样因为非常灵活，业务边界的维护严重依赖开发同学的自觉性，很难长期保持清晰的业务边界

gradle source set:

- 优势：gradle 原生支持，同一个业务模块代码和资源一起管理；
- 劣势：代码编译上不隔离

gradle module:

- 优势：不同业务间, 代码编译隔离，资源独立；
- 劣势：需要的拆分成本非常高

**1.3 拆分方式**

directory + gradle source set + package
```python
main/D
user/D
setting/D
commercial/D
social/D
audio/D
video/D
    audience/D
        m_share
        m_finish
        m_notice
        m_float_activity
        m_redpacket
        m_entry
    creator/D
        m_beauty
        m_activities
        m_entry
    link/D
        m_audience
            src
                com.meelive.ingkee.link.ui.audience/Pacakge
                    IPKView.java
                    IPKPresenter.java
                    presenter
                    view
                    util
            res
                layout
                drawable
                values
        m_creator
            src
                com.meelive.ingkee.link.ui.creator/Pacakge
                    IPKView.java
                    IPKPresenter.java
            res
                layout
                drawable
                values
        m_model
            src
                com.meelive.ingkee.link/Pacakge
                    api
                    bean
                    conn
                    LinkManager.java
                com.meelive.ingkee.link.ui.widget/Pacakge
                    PKAnimation.java
                    PKView.java
            res
                layout
                anim
    multilink/D
        m_audience
        m_creator
        m_model
    m_privacy
    m_channel
    m_publiclive
    m_widget
    m_chatter
    m_top
    m_usermanager
    m_userinfo
    m_player
    m_connection
    m_pk
    m_record
    m_chat
game/D
```

1. 每个独立的业务的都是一个独立的gradle source set（没有编译隔离的‘fake module’）。业务自身的代码和资源，业务自己管理和维护
2. 最顶层是Directory，将业务根据大块需求划分开。directory层次不宜太深，最好不要超过2层
3. 中间层次为 fake module （m_business）是每一个具体的，独立的业务。
4. 每个业务模块不能太大，一般情况下拆分粒度越细越好，单业务模块最大不超过3000行代码
5. 代码量和业务模块没有必然关系，只要足够独立，只有一个类也可以是一个业务模块
6. 单个业务内部代码使用java package（语言级别的命名空间）管理
7. 业务内部的代码设计，推荐 M 和 UI（vc，vp，vvm...）分开。限制VP层的复用性，一个业务可以有多套VP，共用同一个M。也可以存在业务内部的共有widget和工具类。但是反对将widget和util职责扩张到满足业务外需求的倾向
8. VP的复用的可能性，仅仅是临时的，在业务有变化的情况下，优先考虑复制VP，而不是扩展。VP的复制和业务内widget, util的复用不冲突。清晰的职责和边界要比DRY原则更重要
9. 拆分是一件付出少，收益大，影响长远的事情

---

## 二. 业务模块内架构模式
1. mv* 的各种变种中，V和* 是和业务强绑定的，不应该追求V* 的复用性（下面假设采用MVP模式）
2. 业务模块间有复用性的是M
3. M 包括状态管理和数据类型定义
4. MV* 模式对大一些的项目而言，是业务模块级别的模式，不是App整体级别的设计模式
5. Android 上传播相对广泛，相对成熟的是MVP。 对MVP的理解，请[查看wiki](http://wiki.inkept.cn/pages/viewpage.action?pageId=38546022)

---

## 三. 模块之间的关系

**3.1 最简单的情况，每个业务模块相对独立**

![arch_lin_full-context-center.png](https://raw.githubusercontent.com/weixinfree/PickRepo/master/images/arch_lin_full-context-center-dep.png)

1. 每个业务模块内部的设计其实是自由的
2. 推荐业务模块使用MVP模式

**3.2 业务模块间需要通信，但是通信模式相对简单**

![arch_lin_full-context-center.png](https://raw.githubusercontent.com/weixinfree/PickRepo/master/images/arch_lin_full-context-center.png)

通信方式：

1. 启动一个新的页面，或者对话框 （可以通过Route解耦依赖）
2. 从别的业务模块获取数据

注意：

1. M 之间尽量不要相互依赖
2. V, P 可以直接依赖不同业务模块的M，但是一个业务模块的V,P 不能依赖其它模块的V,P
3. 每个业务V,P都可以依赖多个M

**3.3 业务模块之间需要大量通信，或者深层次的通信，上面的模式不再满足需要**

![arch_lin_full-context.png](https://raw.githubusercontent.com/weixinfree/PickRepo/master/images/arch_lin_full-context.png)

1. EventCenter 可以类比为进程内的 MessageQueue；ServiceCenter类比为进程内的RPC。相关的适用范围和使用场景也是类似的
2. 所有的业务模块都可以依赖EventCenter 和 ServiceCenter，但不存在反向依赖
3. EventCenter 和 ServiceCenter 本身没有任何业务，只提供转发

**EventCenter**

1. EventCenter 模式类似App全局的goto，要严格限制使用场景
2. Event通信只适用于业务模块间的通信，限制模块内部使用
3. EventCenter适合于使用 Observer模式的场景

**ServiceCenter**

1. 每个业务模块都可以从ServiceCenter中获取其它模块提供的服务
2. 一个业务模块也可以提供Service给其它模块使用
3. Service接口的设计要遵循最少信息原则，合理封装，不要暴漏太多的实现细节


**EventCenter和ServiceCenter通信的数据结构定义应该属于谁的职责？**

模式1:

1. 将 Event/Struct，Serice接口 放在全局可见的EventCenter 和 ServiceCenter中
2. 优势是相对简单，劣势是通信一多，全局的定义又成了一个非法之地，不能有效的进行管理和拆分

模式2:

1. 将模块的接口层单独抽取出来作为一个独立的module
2. 接口层对于EventCenter是 Event定义
3. 接口层对于ServiceCenter 是 Service接口定义和接口需要的数据结构定义
4. 优势是清晰的依赖，清晰的职责
5. 使用步骤上相对麻烦，对开发同学的自觉性要求也更高一些

推荐优先使用模式1，毕竟绝大部分业务都没有那么复杂


**3.4 业务模块间关系非常复杂，需要查询/交换大量的状态**

![arch_lin_full.png](https://raw.githubusercontent.com/weixinfree/PickRepo/master/images/arch_lin_full.png)

1. 优先考虑使用上面的方式解决，只有整个系统非常复杂，业务模块之间的相互依赖太多，使用统一的状态共享中心明显更有效率的时候，才是合适的使用时机，例如：映客直播间，子业务数量非常大，各个业务模块需要共享很多的状态，使用Context反而会更清晰一点
2. 添加Context角色，Context的角色是上下文信息，维护共享数据
3. Context不一定是单例，但是所有的Context要共享一份实例
4. 应该使用Context，还是直接从其它业务模块的Model查询，还有一个判别标准：当系统需要的非业务数据状态（例如：是否在直播间中，主播liveId，主播uid，是否正在pk，正在连麦，正在播放礼物，是否横屏，软键盘是否弹出）非常多的时候，说明直接查询Model不是特别有效了。这时候使用Context会更合理。非业务数据状态的维护职责不够清晰，反而可以使用Context来解决
5. 从Context查询状态是ok的，Context状态的维护由各个业务共同维护，但是一个状态只能由一个业务模块进行修改，不能多个业务模块同时修改

---

## 四. 分支管理，PR 和 代码review

![git_flow](https://raw.githubusercontent.com/weixinfree/PickRepo/master/images/gitflow.png)

主要是[git flow](https://nvie.com/posts/a-successful-git-branching-model/)模型的稍微改造，和现行分支模型的不同是：

1. 单repo，多分支模式
2. master分支固定
3. feature分支到develop分支需要pull request/merge request，不能直接push
4. pull request/merge request 通过权限收缩，作为代码上线的卡口（静态检测，自动化测试，代码review等）


其它实践：
1. 需要长时间开发的 feature 分支最好定期合并develop分支，避免和develop分支代码差别太大
2. feature分支之间永远不会相互合并
3. feature分支pr之前最好rebase一下

#### git flow介绍
1. [原始英文文章](https://nvie.com/posts/a-successful-git-branching-model/)
2. [阮一峰的翻译](http://www.ruanyifeng.com/blog/2012/07/git.html)
3. [Git工作流指南：Gitflow工作流](http://blog.jobbole.com/76867/)
4. [gitflow workflow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow)