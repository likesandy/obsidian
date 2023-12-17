组件之间就像发动机的各个零件，想要让 React 这台机器运作起来，就要处理好各个零件，也就是各个组件之间的联系，而props 担任的角色就是将每个组件联系起来
props 可以是什么？
- 数据源
- callback
- 组件
- children
## 监听props变化
### 类组件
~~componentWillReceiveProps~~ -> [getDerivedStateFromProps](https://react.dev/reference/react/Component#static-getderivedstatefromprops)
### 函数组件
useEffect
## props childer模式
```jsx
const Children = (props)=> (<div>
    <div>hello, my name is {  props.name } </div>
    <div> { props.mes } </div>
</div>)

function  Container(props) {
    const ContainerProps = {
        name: 'alien',
        mes:'let us learn react'
    }
     return props.children.map(item=>{
        if(React.isValidElement(item)){ // 判断是 react elment  混入 props
            return React.cloneElement(item,{ ...ContainerProps },item.props.children)
        }else if(typeof item === 'function'){
            return item(ContainerProps)
        }else return null
     })
}

const Index = ()=>{
    return <Container>
        <Children />
        { (ContainerProps)=> <Children {...ContainerProps} name={'haha'}  />  }
    </Container>
}
```
