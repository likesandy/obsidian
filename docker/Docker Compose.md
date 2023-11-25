举例：docker 的方式需要手动 docker build 来构建 nest 应用的镜像，然后按顺序使用 docker run 来跑 mysql、redis、nest 容器
（要注意 nest 容器里需要使用宿主机 ip 来访问 mysql、redis 服务）
docker compose 就只需要写一个 `docker-compose.yml`文件，配置多个 `service` 的启动方式和 `depends_on`依赖顺序。
然后 `docker-compose up` 就可以批量按顺序启动一批容器