---
layout: post
title: 解决useEffect重复调用问题
date: 2020-04-07 11:23:19
tags: [React]
categories:
 - web-framework
---
`useEffect`是React hooks中可以让你在函数组件中执行副作用操作的[Effect Hook](https://zh-hans.reactjs.org/docs/hooks-effect.html)。

在React hooks刚出来的时候我也记录过一篇关于[认识 react Hooks](https://seminelee.github.io/2019/03/04/react-hooks/)的。在使用的过程中，经常遇到`useEffect`重复调用的问题，因此借此文总结下。
<!-- more -->

# 1 为什么会出现重复请求的问题？
总结一下原因可能会是：
## 1.1 你没有设置effect依赖参数
比如下面的例子，它在第一次渲染之后和每次更新之后都会执行。
``` js
const [count, setCount] = useState(0)

useEffect(() => {
  document.title = `You clicked ${count} times`;
})
```
这是因为每次重新渲染，都有它自己的 Props and State。每一个组件内的函数（包括事件处理函数，effects，定时器或者API调用等等）会捕获某次渲染中定义的props和state。某种意义上讲，effect 更像是渲染结果的一部分 ——__每个 effect “属于”一次特定的渲染__。

事实上这也正是我们可以在 effect 中获取最新的`count`的值，而不用担心其过期的原因。
如果是没有设置effect依赖参数的原因，在`useEffect`的第二个参数设置好依赖项就可以了。
## 1.2 你设置的依赖频繁变化
有时候我们已经设置了依赖，但是发现还是会无限重复。有可能是你的依赖就是频繁变化的，即在改变状态的方法中用到了状态，比如：
``` js
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1); // 这个 effect 依赖于 `count` state
    }, 1000);
    return () => clearInterval(id);
  }, [count]);

  return <h1>{count}</h1>;
}
```
要解决这个问题，我们可以使用`setState`的[函数式更新](https://zh-hans.reactjs.org/docs/hooks-reference.html#functional-updates)形式。它允许我们指定 state 该 如何 改变而不用引用 当前 state：
``` js
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(c => c + 1); // ✅ 在这不依赖于外部的 `count` 变量
    }, 1000);
    return () => clearInterval(id);
  }, []); // ✅ 我们的 effect 不适用组件作用域中的任何变量

  return <h1>{count}</h1>;
}
```
详细可以看官网的[FAQ](https://zh-hans.reactjs.org/docs/hooks-faq.html#what-can-i-do-if-my-effect-dependencies-change-too-often)
## 1.3 设置的依赖是引用数据类型
其实这也属于第二个原因，如果我们设置的依赖是引用数据类型，我们会发现设置的依赖总是会改变。

比如下面这个例子，打开控制台，会看到至少2次输出。上面也提到了，每次重新渲染，函数组件都有它自己的 Props and State。因此，React在对比时会得出该依赖每次都不相同。即使看起来内容相同，但是每次的引用地址都不一样，即`[] !== []`。
``` js
const [data, setData] = useState([] as any)
useEffect(() => {
  setTimeout(() => {
    setData([])
  }, 100)
}, [])

useEffect(() => {
  setTimeout(() => {
    console.log(data)
  }, 200);
}, [data])
```

我们接下来要详细探讨的第三个原因的解决方法。

> __关于依赖项不要对React撒谎__
> 如果你设置了依赖项，effect中用到的所有组件内的值都要包含在依赖中。这包括props，state，函数 — 组件内的任何东西。解决问题的方法不是移除依赖项。只有依赖项包含了所有effect中使用到的值，React才能知道何时需要运行它。

# 2 函数作为依赖
## 2.1 检查是不是必须把该函数作为依赖
一般建议把不依赖props和state的函数提到你的组件外面，并且把那些仅被effect使用的函数放到effect里面。
``` js
// ✅ Not affected by the data flow
function getFetchUrl(query) {
  return 'https://hn.algolia.com/api/v1/search?query=' + query;
}

function SearchResults() {
  useEffect(() => {
    const url = getFetchUrl('react');
    // ... Fetch data and do something ...
  }, []); // ✅ Deps are OK
  // ...
}
```
## 2.2 useCallback
如果发现你的effect的确需要用到组件内的函数（包括通过props传进来的函数），可以在定义它们的地方用[`useCallback`](https://zh-hans.reactjs.org/docs/hooks-reference.html#usecallback)包一层。为什么要这样做呢？因为这些函数可以访问到props和state，因此它们会参与到数据流中。

`useCallback`本质上是添加了一层依赖检查。它以另一种方式解决了问题——我们使函数本身只在需要的时候才改变，而不是去掉对函数的依赖。
``` js
function SearchResults() {
  const [query, setQuery] = useState('react');

  // ✅ Preserves identity until query changes
  const getFetchUrl = useCallback(() => {
    return 'https://hn.algolia.com/api/v1/search?query=' + query;
  }, [query]);  // ✅ Callback deps are OK

  useEffect(() => {
    const url = getFetchUrl();
    // ... Fetch data and do something ...
  }, [getFetchUrl]); // ✅ Effect deps are OK

  // ...
}
```
如果是props传进来的函数，上面的例子中的`getFetchUrl`可以写成下面这样。props传进来的函数可以访问到props和state。把它的定义包裹进 useCallback Hook。这就确保了它不随渲染而改变，除非它自身的依赖发生了改变。
``` js
const getFetchUrl = useCallback(props.fetchData, [query])
```
[`useMemo`](https://zh-hans.reactjs.org/docs/hooks-reference.html#usememo)可以做类似的事情以避免非必要的渲染。`useCallback(fn, deps)` 相当于 `useMemo(() => fn, deps)`。这里就不再叙述了。
# 3 对象作为依赖
## 3.1 检查是不是必须把对象作为依赖
首先可以检查下是不是必须把该对象作为依赖，比如：
 - 只需要用到该对象的某个非引用类型的属性；
 - 是JSON对象，可以通过`JSON.stringify()`转为字符串传递。子组件再将props传进来的JSON字符串用`JSON.parse()`解析。
## 3.2 useRef
如果上面的方法都无法解决，希望`useRef`可以解决你的问题。

到目前为止，我们知道，每一个组件内的函数（包括事件处理函数，effects，定时器或者API调用等等）会捕获某次渲染中定义的props和state。因此，解决问题的关键就在于，在effect的回调函数里读取最新的值而不是捕获的值，即从过去渲染中的函数里读取未来的props和state。指南中将此形象地比喻成[逆潮而动](https://overreacted.io/zh-hans/a-complete-guide-to-useeffect/#%E9%80%86%E6%BD%AE%E8%80%8C%E5%8A%A8)。

`useRef`就可以做到这一点。不同于effect捕获某次渲染中定义的props和state，`useRef`的`.current`属性就像一个保存一个可变值的“盒子”，可以获取最新的值。而且当 ref 对象内容发生变化时，`useRef`并不会通知你。变更`.current` 属性不会引发组件重新渲染。

1.3中的例子可以改写成这样。打开控制台，可以看到只输出了最新的值`[]`。
``` js
const [data, setData] = useState([] as any)
const dataRef = useRef(data)
useEffect(() => {
  setTimeout(() => {
    setData([])
  }, 100);
}, [])

useEffect(() => {
  dataRef.current = data
})

useEffect(() => {
  setTimeout(() => {
    console.log(dataRef.current)
  }, 200);
}, [])
```

# 参考
 - [React官方文档](https://zh-hans.reactjs.org/docs/getting-started.html)
 - [useEffect 完整指南](https://overreacted.io/zh-hans/a-complete-guide-to-useeffect/)
 