类组件的状态比如 state ，context ，props 本质上是存在类组件对应的 fiber 上，包括生命周期比如 componentDidMount ，也是以副作用 effect 形式存在的。那么 Hooks 既然赋予了函数组件如上功能，所以 hooks 本质是离不开函数组件对应的 fiber 的。 hooks 可以作为函数组件本身和函数组件对应的 fiber 之间的沟通桥梁
![[Pasted image 20231213232125.png]]

hooks 对象本质上是主要以三种处理策略存在 React 中：
- `ContextOnlyDispatcher`：防止开发者在函数组件外部调用 hooks ，所以第一种就是报错形态，只要开发者调用了这个形态下的 hooks ，就会抛出异常
- `HooksDispatcherOnMount`：函数组件初始化 mount
- `HooksDispatcherOnUpdate`：函数组件的更新

一个 hooks 对象应该长成这样
```jsx
const HooksDispatcherOnMount = { /* 函数组件初始化用的 hooks */
    useState: mountState,
    useEffect: mountEffect,
    ...
}
const  HooksDispatcherOnUpdate = {/* 函数组件更新用的 hooks */
   useState:updateState,
   useEffect: updateEffect,
   ...
}
const ContextOnlyDispatcher = {  /* 当hooks不是函数内部调用的时候，调用这个hooks对象下的hooks，所以报错。 */
   useEffect: throwInvalidHookError,
   useState: throwInvalidHookError,
   ...
}
```

## 函数组件触发
所有函数组件的触发是在 `renderWithHooks` 方法中，在`fiber reconcile`过程中，遇到`FunctionComponent`类型的fiber（函数组件），就会用 `updateFunctionComponent` 更新 `fiber` ，在 `updateFunctionComponent` 内部就会调用 `renderWithHooks`
`react-reconciler/src/ReactFiberHooks.js`
[beginWork源码](https://github.com/facebook/react/blob/493610f299ddf7d06e147e60dc4f2b97482982d2/packages/react-reconciler/src/ReactFiberBeginWork.js#L4115)
```jsx
function beginWork() {
  switch (workInProgress.tag) {
    case FunctionComponent: {
      return updateFunctionComponent()
    }
  }
}

function updateFunctionComponent() {
  renderWithHooks()
  reconcileChildren()
}

function renderWithHooks() {
  // 当前正在工作的fiber树
  currentlyRenderingFiber = workInProgress
  //memoizedState：存放hooks的信息
  workInProgress.memoizedState = null
  // 存放每个 useEffect/useLayoutEffect 产生的副作用组成的链表，commit 阶段更新这些副作用
  workInProgress.updateQueue = null

  // 判断是初始化组件还是更新组件
  ReactCurrentDispatcher.current =
    current === null || current.memoizedState === null
      ? HooksDispatcherOnMount
      : HooksDispatcherOnUpdate

  // 执行函数组件，所有的hooks依次执行
  let children = Component(props, secondArg)

  finishRenderingHooks()
}

function finishRenderingHooks() {
  // 将 hooks 变为 ContextOnlyDispatcher 对象(防止开发者在函数组件外部调用 hooks)
  // React 就是通过赋予 current 不同的 hooks 对象达到监控 hooks 是否在函数组件内部调用
  ReactCurrentDispatcher.current = ContextOnlyDispatcher
}
```
## hooks 如何和 fiber 建立起关系
每一个hooks 初始化都会执行 mountWorkInProgressHook
`react-reconciler/src/ReactFiberHooks.js`
```jsx
```