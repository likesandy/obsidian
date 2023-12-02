![[Pasted image 20231202183300.png]]
![[Pasted image 20231202174113.png]]
## babel解析jsx
- @babel/plugin-syntax-jsx
- @babel/plugin-transform-react-jsx
### Automatic
新版本react已经不需要引入createElement，在运行时plugin-syntax-jsx插件已经给文件注入了_jsxRuntime api，不过需要配置babel
![[Pasted image 20231202180426.png]]
```.babelrc
"presets": [    
    ["@babel/preset-react",{
    "runtime": "automatic"
    }]     
],
```
### Classic
在经典模式下，使用 JSX 的文件需要引入 React ，不然就会报错
![[Pasted image 20231202180453.png]]

