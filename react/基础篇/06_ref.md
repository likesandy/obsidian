>react-reconciler/src/ReactFiberCommitWork.js
```jsx
function safelyDetachRef(current: Fiber) {
  const ref = current.ref
  // function/string 放松
  if (typeof ref === 'function') {
    refCleanup()
  } else {
    // 对象方式
    ref.current = null
  }
}
```
> react-reconciler/src/ReactFiberCommitWork.js
```jsx
function commitAttachRef(finishedWork: Fiber) {
  const ref = finishedWork.ref
  if (ref) {
    let instanceToUse
    switch (finishedWork.tag) {
      case HostComponent: //元素节点 获取元素
        instanceToUse = getPublicInstance(instance)
        break
      default: // 类组件直接使用实例
        instanceToUse = instance
    }
    if (typeof ref === 'function') {
      ref(instanceToUse) // function 和 字符串获取方式
    } else {
      ref.current = instanceToUse // ref对象方式
    }
  }
}
```
当 ref 属性是一个字符串的时候，React 会自动绑定一个函数，用来处理 ref 逻辑
>react-reconciler/src/ReactChildFiber.js
```jsx
const ref = function (value) {
  const refs = resolvedInst.refs
  if (value === null) {
    delete refs[stringRef]
  } else {
    refs[stringRef] = value
  }
}
```
当这样绑定ref="node"，会被绑定在组件实例的refs属性下面

**safelyDetachRef调用时机**
>react-reconciler/src/ReactFiberCommitWork.js.js
```jsx
function commitMutationEffectsOnFiber(finishedWork: Fiber) {
  const current = finishedWork.alternate

  switch (finishedWork.tag) {
    case ClassComponent: {
      if (flags & Ref) {
        if (current) {
          safelyDetachRef(current, current.return)
        }
      }
    }
  }
}
```
**commitAttachRef调用时机**
>react-reconciler/src/ReactFiberCommitWork.js
```jsx
function commitLayoutEffectOnFiber(finishedWork: FiberRoot) {
  const flags = finishedWork.flags

  switch (finishedWork.tag) {
    case ClassComponent: {
      if (flags & Ref) {
        safelyAttachRef(finishedWork, finishedWork.return)
      }
    }
  }
}
```
从上可以清晰的看到只有含有 `Ref` tag 的时候，才会执行更新 ref
那么是每一次更新都会打 `Ref` tag 吗？
>react-reconciler/src/ReactFiberBeginWork.js
```jsx
function markRef(current: Fiber | null, workInProgress: Fiber) {
  const ref = workInProgress.ref
  if (
    (current === null && ref !== null) || // 初始化的时候
    (current !== null && current.ref !== ref) // 更新的时候
  ) {
    // Schedule a Ref effect
    workInProgress.flags |= Ref
    workInProgress.flags |= RefStatic
  }
}
```
每一次更新的时候，都给 ref 赋值了新的函数，那么 `markRef` 中就会判断成 `current.ref !== ref`，所以就会重新打 Ref 标签，那么在 commit 阶段，就会更新 ref 执行 ref 回调函数了
```jsx
import { Component } from 'react'

export default class App extends Component {
  count = 0
  render() {
    return (
      <>
        <div
          ref={(node) => {
            this.node = node
            console.log('此时的参数是什么：', this.node)
          }}
        >
          ref元素节点
        </div>
        <button onClick={() => this.setState({ count: this.count++ })}>{this.count}</button>
      </>
    )
  }
}
```
![[Pasted image 20231207222559.png]]
如果ref 指向相同的函数，就不会打 Ref 标签，不会更新 ref 逻辑
当你使用箭头函数时，箭头函数会捕获其所在上下文的`this`，因此它会使用外部的`this`，即组件实例的`this`。这样做的结果是，每次渲染时，都会传递相同的函数引用给`ref`属性，因此不会触发额外的`ref`回调
```jsx
import { Component } from 'react'

export default class Index extends Component {
  state = { num: 0 }
  node = null
  getDom = (node) => {
    this.node = node
    console.log('此时的参数是什么：', this.node)
  }
  render() {
    return (
      <div>
        <div ref={this.getDom}>ref元素节点</div>
        <button onClick={() => this.setState({ num: this.state.num + 1 })}>点击</button>
      </div>
    )
  }
}
```
## 卸载 ref
当组件或者元素卸载的时候，ref 的处理逻辑是怎么样的
被卸载的 fiber 会被打成 `Deletion` effect tag ，然后在 commit 阶段会进行 commitDeletion 流程,
对于有 ref 标记的 ClassComponent （类组件） 和 HostComponent （元素），会统一走 `safelyDetachRef` 流程
>react-reconciler/src/ReactFiberCommitWork.js
```jsx

```