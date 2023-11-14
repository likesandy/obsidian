Docker 是一种容器技术，它可以在操作系统上创建多个相互隔离的容器。容器内独立安装软件、运行服务
![[Pasted image 20231114220450.png]]
但是，这个容器和宿主机还是有关联的，比如可以把宿主机的端口映射到容器内的端口、宿主机某个目录挂载到容器内的目录
![[Pasted image 20231114220528.png]]
比如映射了 3000 端口，那容器内 3000 端口的服务，就可以在宿主机的 3000 端口访问了

比如挂载了 /aaa 到容器的 /bbb/ccc，那容器内读写 /bbb/ccc 目录的时候，改的就是宿主机的 /aaa 目录，反过来，改宿主机 /aaa 目录，容器内的 /bbb/ccc 也会改，这俩同一个

这分别叫做端口映射、数据卷（volume）挂载

这个容器是通过镜像起来的，通过 docker run image-name

比如：
```shell
# 通过 xxx-image 镜像跑起来一个叫做 xxx-container 的容器
# -p 指定端口映射，映射宿主机的 3000 到容器的 3000 端口
# -v 指定数据卷挂载，挂载宿主机的 /aaa 到容器的 /bbb/ccc 目
docker run -p 3000:3000 -v /aaa:/bbb/ccc --name xxx-container xxx-image
```
这个镜像是通过 Dockerfile 经过 build 产生的
也就是这样的流程：
![[Pasted image 20231114220754.png]]
一般在项目里维护 [[DockeFile]] ，然后执行 docker build 构建出镜像、push 到镜像仓库，部署的时候 pull 下来用 docker run 跑起来

基本 CI/CD 也是这样的流程：
- CI 的时候 git clone 项目，根据 dockerfile 构建出镜像，打上 tag，push 到仓库
- CD 的时候把打 tag 的镜像下下来，docker run 跑起来
这个 Dockerfile 是在项目里维护的，虽然 CI/CD 流程不用自己搞，但是 Dockefile 还是要开发者自己写的

