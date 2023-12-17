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
[mountWorkInProgressHook源码](https://github.com/facebook/react/blob/493610f299ddf7d06e147e60dc4f2b97482982d2/packages/react-reconciler/src/ReactFiberHooks.js#L952)
```jsx
function mountWorkInProgressHook() {
  // 每一个 hooks 执行都会产生一个 hook 对象
  const hook = {
    // hook信息
    memoizedState: null,
    // 初始化state
    baseState: null,
    // 更新队列，用于commit阶段更新
    baseQueue: null,
    // 记录更新的各种状态
    queue: null,
    // 每一个 hook按照顺序通过 next 和下一个 hook 建立起关联（链表）
    next: null,
  }

  // 只有一个hook
  if (workInProgressHook === null) {
    currentlyRenderingFiber.memoizedState = workInProgressHook = hook
  } else {
    //  有多个hook
    workInProgressHook = workInProgressHook.next = hook
  }

  return workInProgressHook
}
```

假设在一个组件中这么写
```jsx
import React from 'react'

export default function Index(){
    const [ number,setNumber ] = React.useState(0) // 第一个hooks
    const [ num, setNum ] = React.useState(1)      // 第二个hooks
    const dom = React.useRef(null)                 // 第三个hooks
    React.useEffect(()=>{                          // 第四个hooks
        console.log(dom.current)
    },[])
    return <div ref={dom} >
        <div onClick={()=> setNumber(number + 1 ) } > { number } </div>
        <div onClick={()=> setNum(num + 1) } > { num }</div>
    </div>
}
```
![[Pasted image 20231214233208.png]]
## hooks更新
更新 hooks 逻辑和之前[[fiber]]章节中讲的双缓冲树更新差不多，会首先取出 `workInProgres.alternate `里面对应的 hook ，然后根据之前的 hooks 复制一份，形成新的 hooks 链表关系。这个过程中解释了一个问题，就是**hooks 规则，hooks 为什么要通常放在顶部，hooks 不能写在 if 条件语句中**，因为在更新过程中，如果通过 if 条件语句，增加或者删除 hooks，在复用 hooks 过程中，会产生复用 hooks 状态和当前 hooks 不一致的问题。举一个例子
将第一个 hooks 变成条件判断形式，具体如下：
```jsx
import React from 'react'

export default function Index({ showNumber }) {
  let number, setNumber
  showNumber && ([number, setNumber] = React.useState(0)) // 第一个hooks
  const [num, setNum] = React.useState(1) // 第二个hooks
  const dom = React.useRef(null) // 第三个hooks
  React.useEffect(() => {
    // 第四个hooks
    console.log(dom.current)
  }, [])
  return (
    <div ref={dom}>
      <div onClick={() => setNumber(number + 1)}> {number} </div>
      <div onClick={() => setNum(num + 1)}> {num}</div>
    </div>
  )
}
```
第一次渲染时候 `showNumber = true` 那么第一个 hooks 会渲染，第二次渲染时候，父组件将 showNumber 设置为 false ，那么第一个 hooks 将不执行，那么更新逻辑会变成这样
![[Pasted image 20231215222254.png]]
第二次复用时候已经发现 hooks 类型不同 `useState !== useRef` ，那么已经直接报错了。所以开发的时候一定注意 hooks 顺序一致性
![[Pasted image 20231215213805.png]]
## dispatchAction
`useState` 和 `useReducer` 原理大同小异，本质上都是触发更新的函数都是 `dispatchAction`
```jsx
const [ number,setNumber ] = React.useState(0)
```
setNumber的本质就是dispathAction，具体看看useState做了什么
`react-reconciler/src/ReactFiberHooks.js`
[mountState源码](https://github.com/facebook/react/blob/493610f299ddf7d06e147e60dc4f2b97482982d2/packages/react-reconciler/src/ReactFiberHooks.js#L1752)
```jsx
function mountState(initialState) {
  // Impl（实现）
  const hook = mountStateImpl(initialState)
  // 获取更新状态
  const queue = hook.queue
  // dispatchSetState 为更新调度的主要函数
  // 调用dispatch的时候绑定了当前fiber，所以用户触发 setNumber 的时候，能够直观反映出来自哪个 fiber 的更新
  const dispatch = dispatchSetState.bind(
    null,
    currentlyRenderingFiber,
    queue
  )
  queue.dispatch = dispatch;
  return [hook.memoizedState, dispatch]
}

function mountStateImpl(initialState) {
  // 如果initialState是函数，获取函数的返回值
  if (typeof initialState === 'function') {
    initialState = initialState();
  }
  // 绑定hook信息，后续返回memoizedState
  hook.memoizedState = hook.baseState = initialState
  // 记录更新的各种状态
  const queue = hook.queue = { ...};
  return hook
}

function dispatchSetState() {
  // 创建一个update队列
  const update = {}

  // 当前fiber正在更新
  if (isRenderPhaseUpdate(fiber)) {
    // 每一次调用dispatchSetState（比如触发 setNumber ）都会先创建一个 update ，然后把它放入待更新 pending 队列中(批量更新)
    const pending = queue.pending;
    // 第一个待更新任务
    if (pending === null) {
      update.next = update;
      // 已经有带更新任务
    } else {
      update.next = pending.next;
      pending.next = update;
    }
    // 当前 fiber 没有更新任务
  } else {
    // aciotn是函数则执行action，否则返回action 比如：setNumber(number=>number+1)
    const lastRenderedReducer = queue.lastRenderedReducer = typeof action === 'function' ? action(state) : action;
    // 当前state
    const currentState = queue.lastRenderedState
    // 上一次的state
    const eagerState = lastRenderedReducer(currentState, action)
    // state相同则不更新
    if (is(currentState, eagerState)) {
      return
    }
    // 发起调度更新
    scheduleUpdateOnFiber(root, fiber, lane);
  }
}
```
改变了state以后等到组件渲染，那么就到了再一次执行 useState，此时走的是更新流程，更新时执行的是`updateState`
`react-reconciler/src/ReactFiberHooks.js`
```jsx
function updateState(initialState) {
  return updateReducer(basicStateReducer, initialState);
}

function updateReducer() {
  const hook = updateWorkInProgressHook();

  // 获取待更新队列
  let baseQueue = hook.baseQueue;
  let first = baseQueue.next
  let update = first;
  // 把待更新的pending队列取出来。合并到 baseQueue
  do {
    if (baseQueue) {
      // 获取初始化的state
      let newState = hook.baseState;
      // 更新state
      newState = reducer(newState, action);
    }
  } while (update !== null && update !== first);

  hook.memoizedState = newState;
  return [hook.memoizedState, action];
}
```
![[Pasted image 20231216164554.png]]
## 处理副作用
在 [[fiber]] 章节讲了，在 render 阶段，实际没有进行真正的 DOM 元素的增加，删除，React 把想要做的不同操作打成不同的 effectTag ，等到commit 阶段，统一处理这些副作用，包括 DOM 元素增删改，执行一些生命周期等。hooks 中的 useEffect 和 useLayoutEffect 也是副作用，接下来以 effect 为例子，看一下 React 是如何处理 useEffect 副作用的
```jsx
function mountEffect(create, deps) {
function mountEffect(create, deps) {
  return mountEffectImpl(create, deps);
}

function mountEffectImpl(create, deps) {
  // 获取当前的hook
  const hook = mountWorkInProgressHook();

  // 判断deps是否存在
  const nextDeps = deps ? deps : null;
  //  PassiveEffect | PassiveStaticEffect
  // 给当前的fiber添加effectTag
  currentlyRenderingFiber.flags |= fiberFlags;

  // 创建一个effect添加到hook上
  // 如果存在多个effect，会形成一个链表，绑定在函数组件 fiber 的 updateQueue 上
  // 因为可能存在多个useEffect
  // HookHasEffect是useEffect 的标识符
  // 对于 useLayoutEffect 第一次更新会打上  HookLayout  的标识符
  // 通过标识符，证明是 useEffect 还是 useLayoutEffect ，接下来 React 会同步处理 useLayoutEffect ，异步处理 useEffect **
  hook.memoizedState = pushEffect(HookHasEffect, destroy, create, nextDeps);
}
```
![[Pasted image 20231216170206.png]]
更新effect
```jsx
function updateEffect(create, deps) {
  updateEffectImpl(PassiveEffect, HookPassive, create, deps);
}

function updateEffectImpl() {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps ?? null

  const effect = hook.memoizedState;

  // 正在执行的hook，因为可能存在多个useEffect
  if (currentHook) {
    if (nextDeps) {
      const prevDeps = effect.deps;
      // 如果deps没有发生变化，更新effect list即可
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        pushEffect(effect);
        return;
      }
    }
  }

  // 如果deps依赖项发生改变，赋予 effectTag ，在commit节点，就会再次执行我们的effect
  currentlyRenderingFiber.flags |= PassiveEffect
  hook.memoizedState = pushEffect(HookHasEffect, destroy, undefined, nextDeps);
}
```
## Ref
在 [[06_ref]] 章节详细介绍过，useRef 就是创建并维护一个 ref 原始对象。用于获取原生 DOM 或者组件实例，或者保存一些状态等

创建ref
```jsx
function mountRef(initialValue) {
  const hook = mountWorkInProgressHook()

  const ref = { current: initialValue };
  hook.memoizedState = ref;
  return ref;
}
```
更新ref
```jsx
function updateRef(initialValue) {
  const hook = updateWorkInProgressHook();

  return hook.memoizedState;
}
```
## useMemo
创建
```jsx
function mountMemo(nextCreate, deps) {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps ?? null

  // 执行函数，缓存函数结果
  const nextValue = nextCreate();
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}
```
更新
```jsx
function updateMemo(nextCreate, deps) {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps ?? null

  const prevState = hook.memoizedState;
  if (nextDeps) {
    const prevDeps = prevState[1]
    // 如果deps没有变化，直接返回上一次的值
    if (areHookInputsEqual(nextDeps, prevDeps)) {
      return prevState[0];
    }
  }
  // 如果deps变化了，重新计算值
  const nextValue = nextCreate();
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}
```
