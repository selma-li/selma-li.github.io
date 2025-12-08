---
layout: post
title: 认识 react Hooks
date: 2019-03-04 11:23:19
tags: [React]
categories:
 - web-framework
---
Hooks是React v16.7.0-alpha中加入的新特性。Hook是一种特殊的函数，允许您“钩入”React特性，让你在class以外使用state和其他React特性。
总结下来，Hooks主要有三种Hook，它们分别为我们带来了新的特性：
<!-- more -->

# state Hook 让你可以在函数组件中使用state
Class有一定的学习成本，你必须了解this如何在JavaScript中工作，类也为今天的工具带来了不少的issue。比如，classes不能很好的被minify。
而state Hook 让你可以在函数组件中使用state，从而避免以上的问题。

用过React的同学可能知道“无状态组件”是这样的：
``` js
const Example = (props) => {
  // You can use Hooks here!
  return <div />;
}
```
之前将它说成“无状态组件”，主要是因为无法像在class里一样使用state。
而state Hook可以让你在class以外使用state和其他React特性，现在你可以叫它做函数组件，而不是“无状态组件”了。代码如下：
``` js
import React, { useState } from 'react';

function Example() {
  // 声明一个新的状态变量count
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>
        Click me
      </button>
    </div>
  );
}
```
在这段代码中，我们引入了`useState`这个Hook。先将状态初始化为 `{ count: 0 }`, 然后当用户点击含有`this.setState()`的回调函数时，我们计数，改变这个状态。
__这样写的好处是什么呢？__ 这样，我们就不用写class，也避免了一切class所带来的问题。

`useState`就是一种Hook，我们继续来看其他Hook。

# Effect Hook 让你可以根据相关部分将一个组件分割成更小的函数
然而随着开发的进展组件会变得越来越大、越来越混乱，每个生命周期钩子中都包含了一堆互不相关的逻辑。最终的结果是强相关的代码被分离，反而是不相关的代码被组合在了一起。这显然会导致大量错误。
Effect Hook 让你可以根据相关部分将一个组件分割成更小的函数，而不是根据生命周期，从而避免以上问题。
## 挂载和更新时调用effect
在很多时候，我们想要执行相同的 side effect，不管组件是刚刚挂载，或是刚刚更新。从概念上讲，我们想要它在每次 render 之后执行。
我们在刚刚的例子中再引入`useEffect`Hook，如下：
``` js
import { useState, useEffect } from 'react';

function Example() {
  const [count, setCount] = useState(0);

  // 与 componentDidMount 和 componentDidUpdate 类似:
  useEffect(() => {
    // 通过浏览器自带的 API 更新页面标题
    document.title = `You clicked ${count} times`;
  });

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>
        Click me
      </button>
    </div>
  );
}
```
在这段代码中，每一次更新`count`，我们都会通过`document.title`更新页面标题。`useEffect`中的代码会在第一次 render 和 之后的每次 update 后运行。就相当于在`componentDidMount`、`componentDidUpdate`中写一样。
__那使用它的好处在哪呢？__ 我们可以想一下，如果用class来写上面的代码，则需要在两个生命周期中重复这段代码，如下：
``` js
class Example extends React.Component {
  // ...
  componentDidMount() {
    document.title = `You clicked ${this.state.count} times`;
  }

  componentDidUpdate() {
    document.title = `You clicked ${this.state.count} times`;
  }
  // ...
}
```
而使用`useEffect`的话，React 会记录下你传给 `useEffect` 的这个方法，然后在进行了 DOM 更新之后调用这个方法，这种写法就避免了重复代码了。

## 卸载时清理effect
在 React class 中，典型的做法是在 `componentDidMount` 里创建订阅，然后在 `componentWillUnmount` 中清除它。
那用Effect Hook要怎么做呢？在`useEffect`的方法中返回执行清理的函数就可以了。看下面的例子：
``` js
import { useState, useEffect } from 'react';

function FriendStatus(props) {
  const [isOnline, setIsOnline] = useState(null);

  function handleStatusChange(status) {
    setIsOnline(status.isOnline);
  }

  useEffect(() => {
    ChatAPI.subscribeToFriendStatus(props.friend.id, handleStatusChange);
    // 明确在这个 effect 之后如何清理它
    return function cleanup() {
      ChatAPI.unsubscribeFromFriendStatus(props.friend.id, handleStatusChange);
    };
  });

  if (isOnline === null) {
    return 'Loading...';
  }
  return isOnline ? 'Online' : 'Offline';
}
```
__那使用它的好处在哪呢？__ 明显，这让我们让添加和移除订阅的逻辑彼此靠近。它们是同一个 effect 的一部分！
__那React在什么时候清理 effect呢？__ React 在每次组件 unmount 的时候执行清理。然而，正如我们之前了解的那样，effect 会在 每次 render 时运行，而不是仅仅运行一次。这也就是为什么 React 也会在下次运行 effect 之后清理上一次 render 中的 effect。

如果有多个Effect时，Effect Hook的优势更明显。可以看下面的例子：
``` js
function FriendStatusWithCounter(props) {
  // 关于计数器的state与effect
  const [count, setCount] = useState(0);
  useEffect(() => {
    document.title = `You clicked ${count} times`;
  });
  // 关于是否在线的state与effect
  const [isOnline, setIsOnline] = useState(null);
  useEffect(() => {
    ChatAPI.subscribeToFriendStatus(props.friend.id, handleStatusChange);
    return () => {
      ChatAPI.unsubscribeFromFriendStatus(props.friend.id, handleStatusChange);
    };
  });

  function handleStatusChange(status) {
    setIsOnline(status.isOnline);
  }
  // ...
}
```
个人觉得比起在class中把逻辑拆分到`componentDidMount`和`componentWillUnmount`，这的确更加清晰了。

# 自定义 Hook 让你跨组件复用stateful logic
之前，跨组件复用stateful logic(包含状态的逻辑)十分困难。[render props](https://react.docschina.org/docs/render-props.html) 和 [高阶组件](https://react.docschina.org/docs/higher-order-components.html)都要求你重新构造你的组件，这可能会非常麻烦。
而自定义 Hook 就是用来解决这个问题的。
我们将上面的例子中的`FriendStatus`改一下，改成能被复用的组件`useFriendStatus`，和复用这个组件的stateful logic的两个组件，`FriendStatus`和`FriendListItem`，如下：
``` js
import { useState, useEffect } from 'react';

function useFriendStatus(friendID) {
  const [isOnline, setIsOnline] = useState(null);

  function handleStatusChange(status) {
    setIsOnline(status.isOnline);
  }

  useEffect(() => {
    ChatAPI.subscribeToFriendStatus(friendID, handleStatusChange);
    return () => {
      ChatAPI.unsubscribeFromFriendStatus(friendID, handleStatusChange);
    };
  });

  return isOnline;
}
```
``` js
function FriendStatus(props) {
  const isOnline = useFriendStatus(props.friend.id);

  if (isOnline === null) {
    return 'Loading...';
  }
  return isOnline ? 'Online' : 'Offline';
}
```
``` js
function FriendListItem(props) {
  const isOnline = useFriendStatus(props.friend.id);
  const liStyle = { color: isOnline ? 'green' : 'black' };

  return (
    <li style={liStyle}>
      {props.friend.name}
    </li>
  );
}
```
__两个组件使用相同的Hook共享状态吗?__ 不。自定义Hook是一种重用有状态逻辑的机制(例如设置订阅和记住当前值)，但是每次使用自定义Hook时，它内部的所有状态和效果都是完全隔离的。
__自定义Hook如何获得隔离状态的?__ 因为我们直接调用`useFriendStatus`，从React的角度来看，我们的组件只调用`useState`和`useEffect`。正如我们之前学到的，我们可以在一个组件中多次调用`useState`和`useEffect`，它们将是完全独立的。

# 更多
这里只简单总结了React Hook几个新特性，更多详情可以看官网
 - [Introducing Hooks](https://react.docschina.org/docs/hooks-intro.html)
 - [Using the State Hook](https://react.docschina.org/docs/hooks-state.html)
 - [Using the Effect Hook](https://react.docschina.org/docs/hooks-effect.html)
 - [Building Your Own Hooks](https://react.docschina.org/docs/hooks-custom.html)
 