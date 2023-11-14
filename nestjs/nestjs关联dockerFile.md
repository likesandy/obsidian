我们思考一个问题[[DockerFile]]是在哪里build的？docker里?
答案是在守护进程 `docker daemon`

命令行工具会和 docker daemon 交互来实现各种功能
比如 docker build 的时候，会把 dockerfile 和它的构建上下文（也就是所在目录）打包发送给 docker daemon 来构建镜像
![[Pasted image 20231113230237.png]]
比如我们会执行这样的命令：
```shell
# . 表示构建上下文的目录
docker build -t name:tag -f filename .
```
而镜像当然是越小越好，所以docker支持你使用`.dockerignore`指定那些文件被忽略（不需要发送给docker daemon）
语法类似于`.gitignoe`
```.dockerignoe
# 忽略所有 md 结尾的文件
*.md
# 不包含README.md文件
!README.md
# 忽略ndoe_modules下的所有文件
node_modules/
# 忽略a.txt b.txt c.txt文件
[a-c].txt
.git/
.DS_Store
.vscode/
.dockerignore
.eslintignore
.eslintrc
.prettierrc
.prettierignore
```
此外，还有一种减小镜像体积的手段：**多阶段构建**

我们会先把源码目录发送到 docker daemon 中执行 `npm run build` 来构建产物，之后再 node ./dist/main.js 把服务跑起来

新建一个nest项目：
```shell
nest new docker-nest
```
编写.dockerignore
```.dockerignore
*.md
node_modules/
.git/
.vscode/
.dockerignore
```
编写 Dockerfile：
```Dockerfile
FROM node:18

WORKDIR /app

COPY package.json .

RUN npm config set registry https://registry.npmmirror.com/

RUN npm install

COPY . .

RUN npm run build

EXPOSE 3000

CMD [ "node", "./dist/main.js" ]
```
执行 docker build：
```shell
# 镜像名：docker
# 标签：nest
# 构建上下文：.(当前目录)
docker build -t docker:nest .
```
然后就可以在 docker desktop 里看到你构建出来的镜像了：
![[Pasted image 20231113233044.png]]
点击 run 把它跑起来：
![[Pasted image 20231113233145.png]]
容器跑成功了：
![[Pasted image 20231113233159.png]]
浏览器访问下也没有问题：
![[Pasted image 20231113233221.png]]
这样我们就用 docker 把我们的 nest 应用跑起来了！
但现在 docker 镜像还是不完美的
查看`FIles`文件
明显，src 等目录就不再需要了，构建的时候需要这些，但运行的时候只需要 dist 目录就可以了
![[Pasted image 20231113233404.png]]
把这些文件包含在内，会让镜像体积变大
那怎么办呢？
构建两次么?
- 第一次构建出 dist 目录
- 第二次再构建出跑 dist/main.js 的镜像。那不是要两个 dockerfile？
确实需要构建两次，但只需要一个 dockerfile 就可以搞定
这需要用到 dockerfile 的**多阶段构建**的语法
```Dockerfile
# as别名： build stage
FROM node:18 as build-stage

WORKDIR /app

COPY package.json .

RUN npm config set registry https://registry.npmmirror.com/

RUN npm install

COPY . .

RUN npm run build

# as别名：production stage

FROM node:18 as production-stage

# 从build-stage镜像中复制/app/dist文件到当前镜像的/app目录下
COPY --from=build-stage /app/dist /app

COPY --from=build-stage /app/package.json /app/package.json

WORKDIR /app

# 只安装生产环境依赖
RUN npm install --production

EXPOSE 3000

CMD ["node", "/app/main.js"]
```
这个生产阶段的镜像就指定容器跑起来执行 node /app/main.js 就好了
执行 docker build
```shell
docker build -t nest:second .
```
把之前的容器停掉，把这个跑起来：
![[Pasted image 20231113234311.png]]
这次用 3003 端口来跑：
![[Pasted image 20231113234347.png]]
![[Pasted image 20231113234406.png]]
浏览器访问一下看看：
![[Pasted image 20231113234417.png]]
nest 服务跑成功了
这时候 app 下就是有 dist 的文件、生产阶段的 node\_modules、package.json 这些文件：
![[Pasted image 20231113234500.png]]
对比下镜像体积，明显看出有减小
![[Pasted image 20231113234541.png]]
有人会说，这个镜像体积依然很大啊
那是因为我们用的是基础版的linux镜像（体积比较大）
我们可以换成`alpine`版的（linux发行版，体积小）
![[Pasted image 20231113235155.png]]
```Dockerfile
FROM node:18.0-alpine3.14 as build-stage

WORKDIR /app

COPY package.json .

RUN npm install

COPY . .

RUN npm run build

# production stage
FROM node:18.0-alpine3.14 as production-stage

COPY --from=build-stage /app/dist /app

COPY --from=build-stage /app/package.json /app/package.json

WORKDIR /app

RUN npm install --production

EXPOSE 3000

CMD ["node", "/app/main.js"]
```
构建镜像
```shell
docker build -t nest:ccc .
```
可以看到构建出来的镜像就非常小了：
![[Pasted image 20231113235432.png]]

> 所以一般情况下，我们都会用多阶段构建 + alpine 基础镜像
