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
