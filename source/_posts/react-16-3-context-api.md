---
title: React 16.3 Context API 实践
date: 2018-03-12 00:00:00
tags:
  - tech
  - react
---

# React 16.3 Context API 实践

## 摘要

本文简单介绍了 Context API 的提出背景、API 设计和用法；之后比较了 React-Redux 的设计；然后提出了一种基于 Context API 的二次封装；最后再将二次封装和 mobx-react 进行了比较。

## 背景

React 项目组成员 [@acdlite](https://github.com/acdlite) 于 2017-12-05 提出关于新 Context API 的 [RFC](https://github.com/reactjs/rfcs/pull/2)。

实际上 Context 在 React 的早期版本中就已经存在，但不能解决当 `shouldComponentUpdate` 返回 `false` 时，组件无法响应 context 的改变的问题。由于 `shouldComponentUpdate` 常用于性能优化，被大量开源库或框架广泛使用，因此原版的 Context 变得十分鸡肋，新的 Context API 很好地解决了这一问题。

## Context API

首先安装 16.3.x 版本的 `react` 及 `react-dom`:

```
$ yarn add react@next react-dom@next
```

来看一个简单的例子：

```js
import React, { createContext } from 'react';
import ReactDOM from 'react-dom';

const { Provider, Consumer } = createContext({});

class Child extends React.Component {

  render() {
    // const { date } = this.props;
    return (
      <div>
        <Consumer>
          // 子组件在任意位置通过 <Consumer> 消费顶层给 <Provider> 传入的 value，
          // 感知 value 的变化来重新渲染该区块。
          {({ date }) => <p>{date}</p>}
        </Consumer>
      </div>
    );
  }

}

class App extends React.Component {

  state = {
    date: '',
  };

  componentDidMount() {
    setInterval(() => {
      // 在父组件中更新状态
      this.setState({ date: new Date().toString() });
    }, 1e3);
  }

  render() {
    return (
      <Provider value={this.state}>
        // 父组件不用给子组件显式传入任何数据
        <Child/>
      </Provider>
    );
  }

}

ReactDOM.render(<App/>, document.querySelector('#root'));
```

使用 Context API 最大的好处就是解决深层嵌套组件层层传递 props 的问题。但这样做也存在一个问题： state 的被保存在 `<App>` 中，更新状态时必须调用 `<App>` 的 `this.setState()`，如果子组件需要更新 `state`，那么需要通过 `<Provider>` 向下传递封装了 `this.setState()` 的回调函数:

```js
<Provider value={{ 
  state: this.state,
  actions: { 
    doSomething(newState) { this.setState(newState); }
  }}}
>
  <Child/>
</Provider>
```

之后，子组件要求改变状态时，在 `<Consumer>` 中调用该回调方法即可：

```js
<Consumer>
  {({ state: { date }, actions }) => 
    <button onClick={() => actions.doSomething(...)}>{date}</button>
  }
</Consumer>
```

另有一个问题是如何实现在 **组件外** 更新状态，让组件也能响应状态变化？

这样的需求通常在应用需要与第三方库交互时会遇到，举一个实际的例子：

Q: 一个 web 应用使用 websocket 做数据交换，我们需要在页面上**实时显示** websocket 连接的延迟：

```js
// ws.js
const ws = io.connect('/');

ws.on('pong', (latency) => {
  // 如何将 latency 渲染到组件里？
});
```

先前例子中将 state 内化的方式显然不可行了，这个时候联想到 Redux，利用它全局 store 的设计，借助 `store.dispatch` 就可以实现上面的需求了。

### Redux/React-Redux

`React-Redux` 是对 React 老版本 Context 的封装，它允许子组件通过 `connect` 方法建立对 store 中状态变化的响应，下面是一个简单的 Redux 应用：

```js
// app.js
import React from 'react';
import { createStore } from 'redux';
import { connect } from 'react-redux';

// 创建一个 reducer 来处理 action
function reducer(state = { date: '' }, action) {
  switch (action.type) {
    case 'UPDATE_DATE':
      return { date: action.date };
    default:
      return state;
  }
}

// 创建一个全局 store 来存储状态
const store = createStore(reducer);

class App extends React.Component {

  componentDidMount() {
    setInterval(() => {
      // 发一个 action 来更新 store
      store.dispatch({ type: 'UPDATE_DATE', date: new Date().toString() });
    }, 1e3);
  }

  render() {
    return (
      <Provider store={store}>
        // 父组件不用给子组件显式传入任何数据
        <Child/>
      </Provider>
    );
  }

}

ReactDOM.render(<App/>, document.querySelector('#root'));
```

```js
// child.js
function mapStateToProps(state) {
  return { date: state.date };
}

// 通过 connect 来感知全局 store 的变化
@connect(mapStateToProps, null)
class Child extends React.Component {

  render() {
    return (
      <div>{this.props.date}</div>
    );
  }

}
```

可以看到在 Redux 的套路中，完成一次 `状态更新` 需要 dispatch 一个 action 到 reducer，这个过程同时牵扯到三个概念，有些复杂；而在 Context API 的套路中，完成一次 `状态更新` 只需要 `setState(...)` 就够了，但单纯的 Context API 无法解决先前提到的 **组件外** 更新状态的问题。

### 对 Context API 的简单封装

下面对 Context API 进行二次封装，让它支持类似 Redux 全局 store 的特性，但用法又比 Redux 更加简单。

```js
// context.js
import React, { createContext } from 'react';

const AppContext = createContext();

// self 是对 <Provider> 组件实例的引用
let self = null;

class Provider extends React.Component {

  state = {};

  constructor(props) {
    super(props);
    self = this;
  }

  render() {
    return (
      <AppContext.Provider value={this.state}>
        {this.props.children}
      </AppContext.Provider>
    );
  }

}

const Consumer = AppContext.Consumer;

function getState() {
  if (self) {
    return self.state;
  } else {
    console.warn('cannot getState() because <Provider> is not initialized');
  }
}

function setState(...args) {
  if (self) {
    self.setState(...args);
  } else {
    console.warn('cannot setState() because <Provider> is not initialized');
  }
}

function createStore() {
  return { getState, setState };
}

export { Provider, Consumer, createStore };
```

用法如下：

```js
import React from 'react';
import ReactDOM from 'react-dom';
import { Provider, Consumer, createStore } from './context';

// 新建一个全局 store
const store = createStore();

class Child extends React.Component {

  render() {
    return (
      <div>
        <p>1. 通过回调参数获取最新状态</p>
        <Consumer>
          {({ date }) => <div>{date}</div>}
        </Consumer>
        <p>2. 通过 store.getState() 获取所有状态</p>
        <Consumer>
          {() => <pre>{JSON.stringify(store.getState(), null, 2)}</pre>}
        </Consumer>
        <p>3. 通过 store.setState() 更新状态</p>
        <Consumer>
          {() => <button onClick={() => store.setState({ foo: new Date().toString() })}>子组件触发状态更新</button>}
        </Consumer>
      </div>
    );
  }

}

class App extends React.Component {

  componentDidMount() {
    // 父组件触发状态更新
    setInterval(() => {
      store.setState({ date: new Date().toString() });
    }, 1e3);
  }

  render() {
    return (
      // 现在 <Provider> 不需要任何参数了
      <Provider>
        <Child/>
      </Provider>
    );
  }

}

ReactDOM.render(<App/>, document.querySelector('#root'));
```

现在在应用的任意位置调用 `store.setState()` 方法，就能更新组件的状态了：

```js
import store from './store';

// ws.js
const ws = io.connect('/');

ws.on('pong', (latency) => {
  // 如何将 latency 渲染到组件里？
  store.setState({ latency });
});
```

### 和 mobx/mobx-react 进行比较

MobX 基于观察者模式，通过 mobx-react 封装后许多地方和 Context API 类似，下面是官方提供的一个例子：

```js
class App extends React.Component {
  render() {
     return (
         <div>
            {this.props.person.name}
            <Observer>
                {() => <div>{this.props.person.name}</div>}
            </Observer>
        </div>
     )
  }
}

const person = observable({ name: "John" })

React.render(<App person={person} />, document.body)
person.name = "Mike" // will cause the Observer region to re-render
```

在 mobx-react 的套路中，组件可以通过 `<Observer>` 消费由 `observable()` 创建出来的对象，直接修改该对象中的键值可以实现组件的重新渲染。

二次封装后的 Context API 相比 mobx-react 用法相近，但 mobx 得益于 setter/getter Hooks 具有更直观的状态改变方式。

### 参考资料

- https://github.com/acdlite/rfcs/blob/new-version-of-context/text/0000-new-version-of-context.md
- https://github.com/reactjs/rfcs/pull/2
- https://github.com/facebook/react/pull/11818
