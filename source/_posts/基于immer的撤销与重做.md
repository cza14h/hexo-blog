---
title: 基于immer的撤销与重做
date: 2021-11-22 23:15:28
tags:
  - frontend
  - react
  - 
headimg: /2021/11/22/基于immer的撤销与重做/banner.jpg
---

## 提要与背景

撤销与重做是可视化编辑器项目类似于[draw.io](https://draw.io)涉及到一个常用的需求, 也是之前旧版本一直没填上的坑. 用户操作的入口和复杂程度都算是比较高的, 所以撤销重做的界定(主要体现在连续的拖拉拽移动, 缩放, 旋转)是一个比较关键的点. 并且对于

这次撤销与重做的核心的选型是因为项目中使用了 `@reduxjs/toolkit` 这个redux官方封装的功能增强版轮子, 轮子里面是使用`immer`作为immutable的实现方式.所以利用了`immer`中记录patch的特性来进行差分回滚.

---

## 功能设计

撤销与重做设计思想类似命令模式, 开发者事先注册好确定的指令, 赋予这些指令功能的同时还能够同步记录一些其他运行时的元数据, 在用户交互(执行命令)的同时, 将命令推入栈中方便程序作后续处理亦或者在满足某些条件的情况下丢弃.

套用到当前的项目业务场景中去,可以归纳为
- 这些注册好的确定的指令就是提供给用户的, 用户交互发生时直接/间接需要调用到的方法. 
- 推入栈的数据记录了可以重现用户操作的数据.

### 注册命令
从现有的项目出发, `redux`的核心是一个个`reducer`,这些`reducer`很好的帮助我们去拆分了这一部分注册的方法, 在 `@reduxjs/tookit`的[官网示例](https://redux-toolkit.js.org/api/createSlice#reducers)上是这样定义的:

>``` javascript
import { createSlice, nanoid } from '@reduxjs/toolkit'

const todosSlice = createSlice({
  name: 'todos',
  initialState: [],
  reducers: {
    addTodo: {
      reducer: (state, action) => {
        state.push(action.payload)
      },
      prepare: (text) => {
        const id = nanoid()
        return { payload: { id, text } }
      },
    },
  },
})
>```


我们可以把实现`addTodo`方法看做成一个命令的注册, 我们*约定*在项目中所有提交了`addTodo`这个`action`的行为都将被视作用户的操作行为. 


### 记录数据

我们已经约定了`reducer`作为一条条命令作为用户行为的底层, 而来自用户的行为都将被推入历史记录栈,这个推入的动作可以是在`prepare`或`reducer`方法内 (用于捕获入参, 适用于比较容易提供逆向方法的场景), 或者借助于`immer`来记录变化补丁`patch`的**额外回调**中.



