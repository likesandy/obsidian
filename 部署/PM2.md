## 前言
在生产环境下，我们不会直接跑`node`，而是通过`pm2`来跑
为什么要用pm2？它解决了什么问题？

如果你的 node 应用跑的时候突然抛了个错，崩溃了，是不是需要重新跑起来？
这时候是不是就需要另一个进程来自动做重启这件事情？

node 应用的日志默认输出在控制台，如果想输出到不同的日志文件，是不是可以让另一个进程获取 node 应用的输出，然后写文件来实现？

node是单线程的，而机器是多个 cpu 的，为了充分利用 cpu 的能力，我们会用多个进程来跑 node 应用，这种通用逻辑是不是也可以放到一个单独进程里来实现？

node 运行时的 cpu、内存等资源的占用，是不是需要监控？这时候是不是可以让另一个进程来做？

线上的 node 应用不只是跑起来就行了，还要做**自动重启、日志、多进程、监控**这些事情
而这些事情，都可以用 pm2 来做

## 日志管理
[pm2](https://github.com/Unitech/pm2)(进程管理) 的主要功能就是**进程管理、日志管理、负载均衡、性能监控**这些
首先安装 pm2:
```shell
npm i -g pm2
```
随便跑一个node应用，我这里跑一个Nest应用：
直接 node 跑是这样的，日志打印在控制台：
![[Pasted image 20231115215202.png]]
而用 pm2 的话，就可以这样跑：
```shell
pm2 start ./dist/main.js
```
它会把这个 node 进程跑起来，然后管理起来：
![[Pasted image 20231115215321.png]]
管理起来之后，就有我们上面说的那些功能了，比如自动重启、日志管理、性能监控等
首先查看日志
```shel
pm2 logs
```
![[Pasted image 20231115215458.png]]
可以看到 pm2会把所有进程的日志打印出来，通过前面的**进程id|进程名字**来区分，比如 `0|main`

而且，它会把它写到日志文件里，在 `~/.pm2/logs `下，以进程名-out.log和进程名-error.log分别保存不同进程的日志：
![[Pasted image 20231115215727.png]]
比如 main-out.log 里保存了 main 进程的正常日志，而 main-error.log 里保存了它的报错日志

pm2可以同时跑多个进程，查看日志的时候会看到两个进程的日志
```shell
pm2 logs
```
或者使用(查看指定进程)
```shell
pm2 logs 进程名
pm2 logs 进程id
```
## 进程管理
进程管理的话就是可以手动启动、重启、停止某个进程，而且崩溃了会自动重启，也可以定时自动重启
只需要 pm2 start 的时候带上几个选项就好了：

超过 200M 内存自动重启：
```shell
pm2 start xxx --max-memory-restart 200M
```
从 2s 开始每 3s 重启一次：
```shell
pm2 start xxx --cron-restart "2/3 * * * * *"
```
当文件内容改变自动重启：
```shell
pm2 start xxx --watch
```
不自动重启：
```shell
pm2 start xxx --no-autorestart
```
清空之前的日志
```shell
pm2 flush 进程名 
pm2 flush 进程名|id
```
## 负载均衡
node 应用是单进程的，而为了充分利用多核 cpu，我们会使用多进程来提高性能
node 提供的 cluster 模块就是做这个的，pm2 就是基于这个实现了负载均衡
我们只要启动进程的时候加上 -i num 就是启动 num 个进程做负载均衡的意思
```shell
pm2 start xx -i max
pm2 start xx -i 0
```
![[Pasted image 20231115224245.png]]
我这里跑起了12个进程
跑起来之后，还可以动态调整进程数，通过`pm2 scale`：
```shell
pm2 scale 进程name 数量
```
![[Pasted image 20231115224523.png]]
可以看到 pm2 删除了 8 个，留下了 3 个
```shell
pm2 scale main +3
```
我又加了三个，现在变成了六个
![[Pasted image 20231115224641.png]]
可以动态伸缩进程的数量，pm2 会把请求分配到不同进程上去
这就是负载均衡功能
## 性能监控
执行`pm2 monit`:
```shell
pm2 monit
```
可以看到不同进程的 cpu 和内存占用情况
![[Pasted image 20231115225317.png]]
## 文件配置
当进程多了之后，难道都要手动通过命令行来启动么？
肯定不会每次都敲一遍
pm2 支持配置文件的方式启动多个应用

执行 pm2 ecosystem，会创建一个配置文件：
```shell
pm2 ecosystem
```
apps 部分就是配置应用的，scripts 就是应用的启动路径：
![[Pasted image 20231115225932.png]]
它可以指定的配置非常多，基本就是命令行有啥选项，具体查看[文档](https://pm2.keymetrics.io/docs/usage/application-declaration/)
## 监测仪表板
此外我们还可以访问[pm2](https://pm2.io/)的官网通过仪表板来监控应用
登录注册一个账号，之后把应用和网站关联起来：
![[Pasted image 20231115230819.png]]
![[Pasted image 20231115230905.png]]
可以在线监控你的应用：
![[Pasted image 20231115230935.png]]
## docker部署pm2
查看文档，配置[[DockerFile]]
![[Pasted image 20231115231041.png]]
```DockerFile
# build stage
FROM node:18-alpine3.14 as build-stage

WORKDIR /app

COPY package.json .

RUN npm config set registry https://registry.npmmirror.com/

RUN npm install

COPY . .

RUN npm run build

# production stage

FROM node:18-alpine3.14 as production-stage

COPY --from=build-stage /app/dist /app

COPY --from=build-stage /app/package.json /app/package.json

WORKDIR /app

RUN npm install pm2 -g

ENV PM2_PUBLIC_KEY 18n7y6eco9izgva

ENV PM2_SECRET_KEY hzdckgby39iroui

CMD ["pm2-runtime", "/app/main.js"]
```
build
```shell
docker build -t pm2:nest .
```
