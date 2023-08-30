# docker hub

### 基本操作

- docker ps
- docker image ls
- docker image rm [image id]
- 看日志
  - docker logs [id]
- docker inspect [id] 获取元数据
- 在docker中执行shell
  - docker exec -it [id] bash

### 拉取镜像 

docker pull name:tag

### 镜像操作

- 
- 根据dockerfile创建镜像
  - docker build -t name:tag [dockerfile path]
- 推流
  - docker push name:tag [dest path]
- 运行镜像
  - docker run -it ubuntu bash
    - -it 起交互终端
    - -d 后台运行
    - -p port1:port2  端口映射，本机p1，docker内p2
    - -v dir1:dir2 目录挂载，本机dir1，docker内dir2

### 端口映射