---
title: "对MVP的理解"
date: 2018-11-28T20:07:39+08:00
draft: false
tags: ['mvp','architeture']
---


## M,V,P的职责
**Model:**

- 数据类型定义
- 从数据源获取数据
- 管理业务数据 （更新，维护一致性等等）

**Presenter:**

- Model → View 的Adapter
- 将Model中的业务数据转换为View需要的数据
- 将View的Action 转化为对数据的操作

**View:**

- 负责渲染UI
- 接收用户输入 并转发给Presenter

## MVP之间的关系
![mvp_relations.png](https://raw.githubusercontent.com/weixinfree/PickRepo/master/images/mvp.png)

## MVP的理解和应用
#### 1) Contract 扩展: （Google Android Architecture demo中出现）
- Contract 的意图是: View 和 Presenter 是强绑定的，应该将他们一起定义
- View 和 Presenter 定义上都是 interface, Contract 包含View 和 Presenter，但是一般不应该包含Model

#### 2) Model 
- Model 有一个很重要的职责是维护状态的一致性
- 一个业务可以复用的部分，绝大部分情况下应该是Model
- 不应该追求View 和 Presenter的复用，他们是强绑定于业务，强绑定于UI的
- 很多情况下，客户端的Model非常薄，因为只需要将数据从网络拉取，存储，上传就可以了
- 网络异常的处理应该是Model的职责，失败的细节不应该暴漏给业务
- 数据的有效性验证, 一致性保证应该是Model的职责

#### 3) Model 重要 还是 Presenter 重要 ？
- 针对移动互联网企业的模式，很多情况下，Model层非常薄。就是网络请求get和post （当然也有一部分比较复杂的Model，比如连麦，pk等等 涉及一个长流程上的状态一致性）
- 业务层更多的复杂度在于业务杂糅了太多的东西，另一个相对复杂的方面是炫酷的UI交互
- 业务间的复用应该是Model，UI交互复杂度应该限制在View 和 Presenter内部，不应该对Model产生影响
- 我的理解是Model更重要，把状态管理好了，复杂度会降低非常多，长期的维护性更好。很多的模块，有非常大的View和Presenter，没有合理的状态管理，因为状态混乱带来的问题，导致系统的非本质复杂度非常高

#### 4) 业务逻辑应该放在哪里？
**视情况而定:**一般情况下，靠近数据的逻辑，应该属于Model的职责。靠近UI，交互的业务逻辑，放在presenter中更合适，并没有明确的分割线

#### 5) Widget，Utils属于 M，V，还是 P ？
- MVP属于业务模块级别的设计模式
- 有了MVP，不一定所有的代码都要往MVP三个壳子中套，依然可以有Utils, Widget
- Widget, Utils 最好是业务内部的，尽量不要做成通用的

#### 6) Activity/Fragment 在 MVP 中的角色？
- View 的角色可以使用 Fragment或者组合View 承担
- Activity 可以负责 View ，Presenter 的组装，充当胶水层的角色

#### 7) 典型的错误？
**View 的定义：req**Data**

> 错误原因：方法命名和View 职责不符。better => 'showEmpty, showRecentUsers(List<User> users)'

其它的不断补充

#### 8) 包结构
- 按照业务来组织代码，MVP可以放在一个package下
- 不要按照角色来组织代码，例如 user.model，user.view，user.presenter
- 把业务相关的东西紧密放在一起

#### 9）职责，职责，职责！！！
- 职责划分合理了，设计就成功了一大半
- 超过单个类/单个文件层次的代码编写都需要思考结构上的设计
- 设计不需要尽善尽美，简单清晰最好
- 设计意味着权衡，追求完美是错误的
- 利用合理的假设可以大幅简化设计的复杂度
- 理解和控制复杂度
- 牢记KISS 原则

#### 10) 可测试性
- 在MVP中，因为M不依赖其它任何角色，所以M自身是可以测试的
- V和P相互依赖，所以不做特殊处理的，VP无法单独进行单元测试。
- UI自动化测试相对其它单元测试来说，需要平台环境，运行速度也会非常慢。View的可测试性要求优先级可以降低，甚至放弃
- 将VP抽象为接口，注入空实现/mock实现的方式，就可以保证VP的各自可测试性


#### 11）设计Demo
```java
interface ToDoContract {
    interface View {
        void init(Persenter p)
        void showTodoList(List<Todo> data);
        void toastSaveSuccess();
        void toastSyncFail(String cause);
    }

    interface Presetenter {
        void init(View view)
        void onAddNewTodo(String title, String content);
        void onClickSort(int sortBy)
        void onDelete(int id)
        void onArchive(int id)
        void onSync()
    }
}

class Todo {...}

class TodoModel {
    void save(Todo todo)
    void delete(int id)
    void archive(int id)
    void sync()
    List<Todo> load()
}

interface TodoRepository {

    void save(Todo todo)
    void delete(int id)
    void archive(int id)
    List<Todo> load()
}

class LocalRepositoryImpl implements TodoRepository {}

class CloudRepositoryImpl implements TodoRepository {}
```
#### 12）MVP中的生命周期传递
- 移动端App，有明确的生命周期概念。
- VP角色都应该有生命周期的概念。可以和平台的生命周期绑定
- Model一般来说不应该和生命周期绑定。维护状态大部分情况下，不需要生命周期概念。此外，因为Model既需要维护状态的一致性，同时也需要考虑业务间的复用性, Model有生命周期的话，就要考虑多个业务组件生命周期重叠，交错出现的情况，带来额外的复杂性