## 什么是fiber
fiber（纤维），fiber 在 React 中是最小粒度的执行单元可以理解为fiber = react 的虚拟DOM
fiber包含三层含义：
- 架构：v15数据保存在递归调用栈中，所以被称为`stack reconcile`，v16的reconcile基于fiber节点实现，被称为`fiber reconcile`
- 数据结构：每个fiber节点对应一个`react element`，保存了改组件的类型，对应的DOM节点等信息
- 工作单元：每个fiber节点保存了本次更新中该组件改变的状态、要执行的工作（插入，更新，删除）
## fiber的起源
在`React15`及以前，`Reconciler`采用递归的方式创建虚拟DOM，递归过程是不能中断的。如果组件树的层级很深，递归会占用线程很多时间，造成卡顿
`Reactv16` 为了解决卡顿问题引入了 fiber ，更新 fiber 的过程叫做 `Reconciler`（调和器），每一个 fiber 都可以作为一个执行单元来处理，所以每一个 fiber 可以根据自身的过期时间`expirationTime`（ v17 版本叫做优先级 `lane` ）来判断是否还有空间时间执行更新，如果没有时间更新，就要把主动权交给浏览器去渲染，做一些动画，重排（ reflow ），重绘 repaints 之类的事情，然后等浏览器空余时间，在通过 `scheduler` （调度器），再次恢复执行单元上来，这样就能本质上中断了渲染，提高了用户体验
## 全面认识fiber
### element,fiber,dom三个之间的关系？
element：react在代码层面上的表象，jsx编写的元素结构会被创建成element对象的形式
DOM：元素在浏览器上呈现的表象
fiber：element和DOM的桥梁，每一个类型的element都有一个与之对应的fiber类型，element 变化引起更新流程都是通过 fiber 层面做一次`reconcile`改变，然后对于element，形成新的 DOM 做视图渲染![[Pasted image 20231212212349.png]]
看一下element与fiber之间的对应关系
[源码](https://github.com/facebook/react/blob/0cdfef19b96cc6202d48e0812b5069c286d12b04/packages/react-devtools-shared/src/backend/renderer.js#L204)![[Pasted image 20231212212742.png]]
这里讲两个容易忘的
```jsx
export const HostComponent = 5;           // dom 元素 比如 <div>
export const HostText = 6;                // 文本节点
```
### fiber结构
`react-reconciler/src/ReactFiber.js`
[FiberNode源码](https://github.com/facebook/react/blob/0cdfef19b96cc6202d48e0812b5069c286d12b04/packages/react-reconciler/src/ReactFiber.js#L135)
```jsx
this.tag = tag; // fiber类型
this.key = key; // key reconcile子节点的时候用到
this.elementType = null;  // 组件类型（类，函数，DOM）
this.type = null; // 元素类型
this.stateNode = null;  // 真实dom元素
// Fiber
this.return = null; //  父级fiber
this.child = null;  // 子级fiber
this.sibling = null;  // 兄弟fiber
this.index = 0; // 索引
this.ref = null;  //  ref指向
this.refCleanup = null; // 清除ref
this.pendingProps = pendingProps; // 在一次更新中，代表element创建
this.memoizedProps = null;  // 记录上一次更新完毕后的props
this.updateQueue = null;  //  管理组件状态更新的
this.memoizedState = null;  //  类组件保存state信息，函数组件保存hooks信息，dom元素为null
this.dependencies = null; // 依赖项
this.mode = mode; // fiber树的模式,比如 ConcurrentMode 模式
// Effects
this.flags = NoFlags; // 标识
this.subtreeFlags = NoFlags; // 子树标识
this.deletions = null;  // 要被删除的元素或组件（标记）
this.lanes = NoLanes; // reconciler 优先级
this.childLanes = NoLanes; // 子树的优先级
this.alternate = null;  // 双缓存树，指向缓存的fiber。更新阶段，两颗树互相交替
```
### 每一个fiber如何建立起关联的
每一个 fiber 是通过 return ， child ，sibling 三个属性建立起联系的
比如项目中元素结构是这样的：
```jsx
import { useState } from 'react'

function App() {
  const [count, setCount] = useState(0)

  const handleCount = () => {
    setCount(count + 1)
  }

  return (
    <div>
      我是文本
      <p>我是p标签</p>
      <button onClick={handleCount}>我是button标签{count}</button>
    </div>
  )
}

export default App
```
![[Pasted image 20231212230158.png]]
## Fiber更新机制
### 初始化
**第一步：创建fiberRoot和rootFiber**
- `fiberRoot`：首次构建应用， 创建一个 fiberRoot ，作为整个 React 应用的根节点
- `rootFiber`： 通过 ReactDOM.render 渲染出来的，如上 Index 可以作为一个 rootFiber。一个 React 应用可以有多 ReactDOM.render 创建的 rootFiber ，但是只能有一个 fiberRoot

`fiberRoot`的`current`会指向当前页面上已渲染内容对应`Fiber树`，即`current Fiber树`
`react-reconciler/src/ReactFiberRoot.js`
```jsx
export function createFiberRoot() {
  const fiberRoot = new FiberRootNode()
  const rootFiber = createHostRootFiber();
  fiberRoot.current = rootFiber
  return fiberRoot
}
```
![[Pasted image 20231212230222.png]]
接下来进入`render阶段`，根据组件返回的`JSX`在内存中依次创建`Fiber节点`并连接在一起构建`Fiber树`，被称为`workInProgress Fiber树`。（下图中右侧为内存中构建的树，左侧为页面显示的树）
![[Pasted image 20231212230237.png]]
**第三步：深度`reconcile`子节点，渲染视图**
在新创建的 `alternates` 上，完成整个 fiber 树的遍历，包括 fiber 的创建
![[Pasted image 20231212230657.png]]
最后会以 workInProgress 作为最新的渲染树，fiberRoot 的 current 指针指向 workInProgress 使其变为 current Fiber 树。到此完成初始化流程
![[Pasted image 20231212230912.png]]
### 更新
状态发生更新，接下来会发生什么呢?
- 首先会走如上的逻辑，重新创建一颗 `workInProgresss` 树，复用当前 `current` 树上的 `alternate` ，作为新的 `workInProgress` ，由于初始化 `rootfiber` 有 `alternate` ，所以对于剩余的子节点，React 还需要创建一份，和 `current` 树上的 `fiber` 建立起 `alternate` 关联。渲染完毕后，`workInProgresss` 再次变成 `current` 树
在构建`workInProgress Fiber树`时会尝试复用`current Fiber树`中已有的`Fiber节点`内的属性（这个决定是否复用的过程就是Diff算法）
![[Pasted image 20231212233722.png]]
### 双缓存树
当我们用`canvas`绘制动画，每一帧绘制前都会清除上一帧的画面。
如果当前帧画面计算量比较大，导致清除上一帧画面到绘制当前帧画面之间有较长间隙，就会出现白屏。

为了解决这个问题，我们可以在内存中绘制当前帧动画，绘制完毕后直接用当前帧替换上一帧画面，由于省去了两帧替换间的计算时间，不会出现从白屏到出现画面的闪烁情况。
这种在内存中构建并直接替换的技术叫做**双缓存**

React 用 `workInProgress` 树(内存中构建的树) 和 `current` (当前屏幕上显示内容的fiber树) 来实现更新逻辑。双缓存一个在内存中构建，一个渲染视图，两颗树用 `alternate` 指针相互指向，每次状态更新都会产生新的`workInProgress Fiber树`，通过`current`与`workInProgress`的替换，完成`DOM`更新
## render和commit
render 阶段和 commit 阶段是整个 fiber Reconciler 的核心
在此之前先看一下fiber的遍历开始-`workLoop`
### render
`react-reconciler/src/ReactFiberWorkLoop.js`
[workLoopSync源码](https://github.com/facebook/react/blob/0cdfef19b96cc6202d48e0812b5069c286d12b04/packages/react-reconciler/src/ReactFiberWorkLoop.js#L2040C13-L2040C13)
```jsx
function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}
```
每一个 `fiber` 可以看作一个执行的单元，在`reconcile`过程中，每一个发生更新的 fiber 都会作为一次 `workInProgress` 。那么 `workLoopSync` 就是执行每一个单元的调度器，如果渲染没有被中断，那么 `workLoop` 会遍历一遍 fiber 树

`performUnitOfWork` 包括两个阶段 `beginWork` 和 `completeWork`
`react-reconciler/src/ReactFiberWorkLoop.js`
```jsx
function performUnitOfWork(){
    next = beginWork(current, unitOfWork, renderExpirationTime);
    if (next === null) {
       next = completeUnitOfWork(unitOfWork);
    }
}
```
`beginWork`：是向下`reconcile`的过程。就是由 fiberRoot 按照 child 指针逐层向下`reconcile`，期间会执行函数组件，实例类组件，diff 调和子节点，打不同effectTag
`completeUnitOfWork`：是向上归并的过程，如果有兄弟节点，会返回 sibling兄弟，没有返回 return 父级，一直返回到 fiebrRoot ，期间可以形成effectList，对于初始化流程会创建 DOM ，对于 DOM 元素进行事件收集，处理style，className等。

这么一上一下，构成了整个 fiber 树的调和
#### 向下reconcile `beginWork`
`react-reconciler/src/ReactFiberBeginWork.js`

- 对于组件，执行部分生命周期，执行 render ，得到最新的 children
- 向下遍历调和 children ，复用 oldFiber ( diff 算法)
- 打不同的副作用标签 effectTag ，比如类组件的生命周期，或者元素的增加，删除，更新

**reconcileChildren**
`react-reconciler/src/ReactFiberBeginWork.js`
[reconcileChildren源码](https://github.com/facebook/react/blob/493610f299ddf7d06e147e60dc4f2b97482982d2/packages/react-reconciler/src/ReactFiberBeginWork.js#L325)
```jsx
function reconcileChildren() {
  if (current === null) {
    // 初始化
    workInProgress.child = mountChildFibers(
      workInProgress,
      null,
      nextChildren,
      renderLanes
    )
  } else {
    // 更新阶段，diff Children
    workInProgress.child = reconcileChildFibers(
      workInProgress,
      current.child,
      nextChildren,
      renderLanes
    )
  }
}
```
**EffectTag**
列举几个常用的 effectTag
```jsx
export const Placement = /*             */ 0b0000000000010;  // 插入节点
export const Update = /*                */ 0b0000000000100;  // 更新fiber
export const Deletion = /*              */ 0b0000000001000;  // 删除fiebr
export const Snapshot = /*              */ 0b0000100000000;  // 快照
export const Passive = /*               */ 0b0001000000000;  // useEffect的副作用
export const Callback = /*              */ 0b0000000100000;  // setState的callback
export const Ref = /*                   */ 0b0000010000000;  // ref
```
#### 向上归并 completeUnitOfWork
- 首先 completeUnitOfWork 会将 effectTag 的 Fiber 节点会被保存在一条被称为 effectList 的单向链表中。在 commit 阶段，将不再需要遍历每一个 fiber ，只需要执行更新 effectList 就可以了。
- completeWork 阶段对于组件处理 context ；对于元素标签初始化，会创建真实 DOM ，将子孙 DOM 节点插入刚生成的 DOM 节点中；会触发 diffProperties 处理 props ，比如事件收集，style，className 处理

### commit
ommit 阶段做的事情是：
- 一方面是对一些生命周期和副作用钩子的处理，比如 componentDidMount ，函数组件的 useEffect ，useLayoutEffect
- 另一方面就是在一次更新中，添加节点（ `Placement` ），更新节点（ `Update` ），删除节点（ `Deletion` ），还有就是一些细节的处理，比如 ref 的处理

commit 细分可以分为：
- `Before mutation` 阶段（执行 DOM 操作前）
- `mutation` 阶段（执行 DOM 操作）
- `layout` 阶段（执行 DOM 操作后）
#### Before mutation
`react-reconciler/src/ReactFiberCommitWork.js`
```jsx

```

Before mutation 阶段做的事主要有以下内容：
- 因为 Before mutation 还没修改真实的 DOM ，是获取 DOM 快照的最佳时期，如果是类组件有 getSnapshotBeforeUpdate ，那么会执行这个生命周期。
- 会异步调用 useEffect ，其目的就是防止同步执行时阻塞浏览器做视图渲染
#### Mutation
`react-reconciler/src/ReactFiberCommitWork.js`
```jsx
commitMutationEffects
```

mutation 阶段做的事情有：
- 置空 ref
- 对新增元素，更新元素，删除元素。进行真实的 DOM 操作。

#### Layout
`react-reconciler/src/ReactFiberCommitWork.js`
```jsx
commitLayoutEffects
```

Layout 阶段 DOM 已经更新完毕，Layout 做的事情有：
- commitLayoutEffectOnFiber 对于类组件，会执行生命周期，setState 的callback，对于函数组件会执行 useLayoutEffect 钩子。  
- 如果有 ref ，会重新赋值 ref
