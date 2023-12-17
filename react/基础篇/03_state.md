老掉牙面试题：**state是同步还是异步的**
React 是有多种模式的，基本平时用的都是 legacy 模式下的 React，除了`legacy` 模式，还有 `blocking` 模式和 `concurrent` 模式， blocking 可以视为 concurrent 的优雅降级版本和过渡版本，React 最终目的，不久的未来将以 concurrent 模式作为默认版本，这个模式下会开启一些新功能。
在当前v18版本中concurrent 将作为默认模式
本章节主要还是围绕 legacy 模式下的 state，concurrent模式后续在单独的章节中进行讲解
## 类组件中的 state
[setState(nextState, callback?)](https://react.dev/reference/react/Component#setstate)
- nextState
	- 如果您将一个对象作为 `nextState` 传递，它将被浅层合并到 `this.state` 中 
	- 如果 nextState 是一个函数，那么当前组件的 state 和 props 将作为参数，应返回要浅合并到 `this.state` 中的对象
- callback 为一个函数，React将在更新commit后调用您提供的回调函数

一次事件中触发一次如上 setState ，在 React 底层主要做了那些事
- 首先，setState 会产生当前更新的优先级（老版本用 expirationTime ，新版本用 lane ）
- 接下来 React 会从 fiber Root 根部 fiber 向下调和子节点，调和阶段将对比发生更新的地方，更新对比 expirationTime ，找到发生更新的组件，合并 state，然后触发 render 函数，得到新的 UI 视图层，完成 render 阶段
- 接下来到 commit 阶段，commit 阶段，替换真实 DOM ，完成此次更新流程
- 此时仍然在 commit 阶段，会执行 setState 中 callback 函数（如果有），到此为止完成了一次 setState 全过程

**更新的流程图如下：**
![[Pasted image 20231203172424.png]]
![[Pasted image 20231203172627.png]]
**类组件如何限制 state 更新视图**
1. `pureComponent` 可以对 state 和 props 进行浅比较，如果没有发生变化，那么组件不更新
2. `shouldComponentUpdate` 生命周期可以通过判断前后 state 变化来决定组件需不需要更新，需要更新返回true，否则返回false
### setState原理
如果调用 setState 方法，实际上是 React 底层调用 updater 对象上的 enqueueSetState 方法
1. enqueueSetState方法做了什么？
2. React 底层是如何进行批量更新的？
`react-reconciler/src/ReactFiberClassComponent.js`
```jsx
/**
 * 类组件更新
 * @param inst 
 * @param payload 
 * @param callback 
 */
function enqueueSetState(inst: any, payload: any, callback){
  // 每一次调用setState，react 都会创建一个 update 里面保存了
  const update = createUpdate(lane);
  // 如果存在callack就保存到update里面
  callback && (update.callback = callback) 
  // 把当前的update和当前fiber，放入待更新队列中
  const root = enqueueUpdate(fiber, update, lane);
  // 开始调度更新
  scheduleUpdateOnFiber(root, fiber, lane);
}
```
React 的 `batchUpdate` 批量更新是什么时候加上去的呢？
正常 **state 更新**、**UI 交互**，都离不开用户的事件，比如点击事件，表单输入等，React 是采用事件合成的形式，每一个事件都是由 React 事件系统统一调度的，那么 State 批量更新正是和事件系统息息相关的
`react-dom/src/events/ReactDOMUpdateBatching.js`
```jsx
// 批量更新（将多次状态更新合并为一次）
export function batchedUpdates(fn, a, b) {
  // 开启批量更新
  isInsideEventHandler = true;
  try {
    // 实际执行批量更新的函数
    // 比如在一次点击事件中触发setState,那么它将在这个函数内执行
    return batchedUpdatesImpl(fn, a, b);
  } finally {
    // 完成一次事件，批量更新
    isInsideEventHandler = false;
  }
}
```
举一个例子，如下组件中这么写：
```jsx
import { Component } from 'react'

class App extends Component {
  state = { number: 0 }
  handleClick = () => {
    this.setState({ number: this.state.number + 1 }, () => {
      console.log('callback1', this.state.number)
    })
    console.log(this.state.number)
    this.setState({ number: this.state.number + 1 }, () => {
      console.log('callback2', this.state.number)
    })
    console.log(this.state.number)
    this.setState({ number: this.state.number + 1 }, () => {
      console.log('callback3', this.state.number)
    })
    console.log(this.state.number)
  }
  render() {
    return (
      <div>
        {this.state.number}
        <button onClick={this.handleClick}>number++</button>
      </div>
    )
  }
}

export default App
```
打印结果
![[Pasted image 20231203200122.png]]
如上代码，在整个 React 上下文执行栈中会变成这样
![[Pasted image 20231203200144.png]]
那么，在异步操作里面的批量更新规则会被打破，比如用 promise 或者 setTimeout 在 handleClick 中这么写：
```jsx
import { Component } from 'react'

class App extends Component {
  state = { number: 0 }
  handleClick = () => {
    setTimeout(() => {
      this.setState({ number: this.state.number + 1 }, () => {
        console.log('callback1', this.state.number)
      })
      console.log(this.state.number)
      this.setState({ number: this.state.number + 1 }, () => {
        console.log('callback2', this.state.number)
      })
      console.log(this.state.number)
      this.setState({ number: this.state.number + 1 }, () => {
        console.log('callback3', this.state.number)
      })
      console.log(this.state.number)
    })
  }
  render() {
    return (
      <div>
        {this.state.number}
        <button onClick={this.handleClick}>number++</button>
      </div>
    )
  }
}

export default App
```
打印结果
![[Pasted image 20231203200248.png]]
那么在整个 React 上下文执行栈中就会变成如下图这样:
![[Pasted image 20231203200323.png]]
**那么，如何在如上异步环境下，继续开启批量更新模式呢？**
React-Dom 中提供了批量更新方法 `unstable_batchedUpdates`，可以去手动批量更新，可以将上述 setTimeout 里面的内容做如下修改:
```jsx
import { Component } from 'react'
import { unstable_batchedUpdates } from 'react-dom'

class App extends Component {
  state = { number: 0 }
  handleClick = () => {
    setTimeout(() => {
      unstable_batchedUpdates(() => {
        this.setState({ number: this.state.number + 1 }, () => {
          console.log('callback1', this.state.number)
        })
        console.log(this.state.number)
        this.setState({ number: this.state.number + 1 }, () => {
          console.log('callback2', this.state.number)
        })
        console.log(this.state.number)
        this.setState({ number: this.state.number + 1 }, () => {
          console.log('callback3', this.state.number)
        })
        console.log(this.state.number)
      })
    })
  }
  render() {
    return (
      <div>
        {this.state.number}
        <button onClick={this.handleClick}>number++</button>
      </div>
    )
  }
}

export default App
```
打印结果
![[Pasted image 20231203200555.png]]
在实际工作中，unstable_batchedUpdates 可以用于 Ajax 数据交互之后，合并多次 setState，或者是多次 useState 原因很简单，所有的数据交互都是在异步环境下，如果没有批量更新处理，一次数据交互多次改变 state 会促使视图多次渲染
**那么如何提升更新优先级呢？**
React-dom 提供了 `flushSync` ，flushSync 可以将回调函数中的更新任务，放在一个较高的优先级中。React 设定了很多不同优先级的更新任务。如果一次更新任务在 flushSync 回调函数内部，那么将获得一个较高优先级的更新
>大多数情况下，可以避免使用 `flushSync`

接下来，将上述 `handleClick` 改版如下样子：
```jsx
import { Component } from 'react'
import { flushSync } from 'react-dom'

class App extends Component {
  state = { number: 0 }
  handleClick = () => {
    setTimeout(() => {
      this.setState({ number: 1 }, () => {
        console.log('setTimeout', this.state.number)
      })
    })
    this.setState({ number: 2 }, () => {
      console.log(`callback1`, this.state.number)
    })
    flushSync(() => {
      this.setState({ number: 3 }, () => {
        console.log('flushSync', this.state.number)
      })
    })
    this.setState({ number: 4 }, () => {
      console.log(`callback2`, this.state.number)
    })
  }
  render() {
    return (
      <div>
        {this.state.number}
        <button onClick={this.handleClick}>number++</button>
      </div>
    )
  }
}

export default App
```
打印结果
![[Pasted image 20231203204557.png]]
 ![[Pasted image 20231203205556.png]]
flushSync 在同步条件下，会合并之前的 setState | useState（强制React刷新任何待处理的工作并同步更新DOM）
可以理解成，如果发现了 flushSync ，就会先执行更新，如果之前有未更新的 setState ｜ useState ，就会一起合并了
2 和 3 被批量更新到 3 ，所以 3 先被打印
## 函数组件中的state
[const [state, setState] = useState(initialState)](https://react.dev/reference/react/useState)
- initialState初始值，第一种情况是非函数，将作为 state 初始化的值。 第二种情况是函数，函数的返回值作为 useState 初始化的值
- state：当前状态
- dispatch：改变 state 的函数，更新状态到**不同的值**并触发重新渲染
	- 第一种非函数情况，此时将作为新的值，赋予给 state，作为下一次渲染使用
	- 第二种是函数的情况，是以上一次的state作为参数，函数返回值会作为新的state
**如何监听 state 变化？**
在函数组件中通过useEffect把state作为dep依赖项用于监听state的变化
```jsx
import { useState, useEffect } from 'react'
import { flushSync } from 'react-dom'

function App() {
  const [number, setNumber] = useState(0)
  /* 监听 number 变化 */
  useEffect(() => {
    console.log('监听number变化，此时的number是:  ' + number)
  }, [number])

  const handleClick = () => {
    /** 高优先级更新 **/
    flushSync(() => {
      setNumber(2)
    })
    /* 批量更新 */
    setNumber(1)
    /* 滞后更新 ，批量更新规则被打破 */
    setTimeout(() => {
      setNumber(3)
    })
  }

  return (
    <div>
      <span> {number}</span>
      <button onClick={handleClick}>number++</button>
    </div>
  )
}

export default App
```
打印结果
![[Pasted image 20231203230740.png]]dispatch 更新效果在本次函数执行上下文中，是获取不到最新的 state 值的
把上述demo 如下这么改：
```jsx
import { useState } from 'react'
import { flushSync } from 'react-dom'

function App() {
  const [number, setNumber] = useState(0)

  const handleClick = () => {
    flushSync(() => {
      setNumber(2)
      console.log(number)
    })
    setNumber(1)
    console.log(number)
    setTimeout(() => {
      setNumber(3)
      console.log(number)
    })
  }
  return (
    <div>
      <span> {number}</span>
      <button onClick={handleClick}>number++</button>
    </div>
  )
}

export default App
```
打印结果
![[Pasted image 20231203230951.png]]函数组件更新就是函数的执行，在函数一次执行过程中，函数内部所有变量重新声明，所以改变的 state ，只有在下一次函数组件执行时才会被更新
### useState原理
相同点
- 底层都调用了 `scheduleUpdateOnFiber` 方法，而且事件驱动情况下都有批量更新规则
不同点
- 在不是 pureComponent 组件模式下， setState 不会浅比较两次 state 的值，只要调用 setState，在没有其他优化手段的前提下，就会执行更新。但是 useState 中的 dispatchAction 会默认比较两次 state 是否相同，然后决定是否更新组件
- setState 有专门监听 state 变化的回调函数 callback，可以获取最新state；但是在函数组件中，只能通过 useEffect 来执行 state 变化引起的副作用
- setState 在底层处理逻辑上主要是和老 state 进行合并处理，而 useState 更倾向于重新赋值
