### 自定义镜像

当执行docker build 的时候就会根据dockerfile 配置文件来自动化构建出一个镜像来

比如这样：
```bash
FROM node:latest

WORKDIR /app

COPY . .

RUN npm config set registry <https://registry.npmmirror.com/>

RUN npm install -g http-server

EXPOSE 8080

CMD ["http-server", "-p", "8080"]
```

这些指令的含义如下：
- FROM：基于一个基础镜像来修改
- WORKDIR：指定当前工作目录
- COPY：把容器外的内容复制到容器内
- EXPOSE：声明当前容器要访问的网络端口，比如这里起服务会用到 8080
- RUN：在容器内执行命令
- CMD：容器启动的时候执行的命令

我们先通过 FROM 继承了 node 基础镜像，里面就有 npm、node 这些命令了

通过 WORKDIR 指定当前目录

然后通过 COPY 把 Dockerfile 同级目录下的内容复制到容器内，这里的 . 也就是 /app 目录

之后通过 RUN 执行 npm install，全局安装 http-server

通过 EXPOSE 指定要暴露的端口

CMD 指定容器跑起来之后执行的命令，这里就是执行 http-server 把服务跑起来

把这个文件保存为 Dockerfile，然后在同级添加一个 index.html


然后通过 **docker build** 就可以根据这个 dockerfile 来生成镜像

```bash
# aaa 是镜像名，ccc 是镜像的tag
docker build -t aaa:ccc .
```

FROM 是继承一个基础镜像，看输出也可以看出来，前面都是 node 镜像的内容，会一层层下载下来

最后才是本地的我们添加的那些

这时你在 desktop 的 images 列表里就可以看到这个镜像了：

然后执行 docker run 把这个镜像跑起来，用 desktop 我们就直接点击 run 按钮了：

指定容器名、映射的端口、点击 run：

当你看到日志的时候，说明就成功了

访问 [http://localhost:8888](http://localhost:8888) 就可以看到我们在 html 写的内容了：

在容器内页打印了一条访问日志：

至此，我们写的第一个 dockerfile 和 build 出的第一个镜像就跑成功了！

我们在 files 里看看 /app 下是啥内容：

我们想修改静态文件怎么办呢？

进入容器内改太麻烦，不如把这个 /app 目录设置为挂载点吧

创建Dockerfile2文件

```docker
FROM node:latest

WORKDIR /app

COPY . .

RUN npm config set registry <https://registry.npmmirror.com/>

RUN npm install -g http-server

EXPOSE 8080

# 设置数据卷
VOLUME [ "/app" ]

CMD ["http-server", "-p", "8080"]
```

重新 build 出一个镜像来：

```bash
# -f 指定文件
docker build -t aaa:ddd -f Dockerfile2 .
```

构建完之后再 run 一下这个新镜像：
![[Pasted image 20231113223144.png]]

![[Pasted image 20231113222713.png]]
我把桌面目录作为数据卷挂在到了/app目录下
![[Pasted image 20231113223334.png]]
容器跑起来了以后也可以看到挂载成功了
![[Pasted image 20231113223423.png]]
浏览器访问下：
![[Pasted image 20231113223554.png]]在 inspect 这里也可以看到挂载的目录：
![[Pasted image 20231113223617.png]]
如果在docker中不指定`VOLUME`,我们可以通过docker run的时候通过-v挂载数据卷
但是如果忘记-v，那么会随机生成一个临时的目录作为数据卷挂载上去
![[Pasted image 20231113224541.png]]
设想下，如果你跑了个 mysql 容器，存了很多数据，但是跑容器的时候没指定数据卷
有一天，你把容器删了，所有数据都没了，可不可怕？
> 为了避免这种情况，mysql 的 dockerfile 里是必须声明 volume 的
> 这样就算你没通过 -v 指定数据卷，将来也可以找回数据

在镜像详情可以看到 mysql 的 dockerfile，确实声明了 volume
![[Pasted image 20231113225541.png]]
这样就能保证数据不丢失
## 使用 ARG 增加构建灵活性
我们写一个 test.js
```js
console.log(process.env.aaa);
console.log(process.env.bbb);
```

使用 ARG 声明构建参数，使用 ${xxx} 来取
然后用 ENV 声明环境变量
dockerfile 内换行使用 `\`
之后构建的时候传入构建参数：

我们来写一个DockerFIle
```Dockerfile
FROM node:18-alpine3.14

ARG aaa
ARG bbb

WORKDIR /app

COPY ./test.js .

ENV aaa=${aaa} \
    bbb=${bbb}

CMD ["node", "/app/test.js"]
```
构建镜像
```shell
docker build --build-arg aaa=3 --build-arg bbb=4 -t arg-test .
```
通过 `--build-arg xxx=yyy` 传入 ARG 参数的值
