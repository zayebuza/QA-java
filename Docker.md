# Docker    -- - - - - - 待完善

#### 什么是Docker？

​	Docker是一种容器，想要了解什么是Docker，需要先从容器开始说起

什么是容器?

​	容器就是将软件打包成标准化单元，以用户开发，交付和部署。通俗的说就是容器就是一个存放东西的地方。。。



#### Docker的基本概念

​	镜像（Image）

​	容器（Container）

​	仓库（Repository）

#### Docker的特点





#### Docker的日常使用

##### 	使用 Redis的Docker容器

查找Docker Hub上的redis镜像
docker search  redis
这里我们拉取官方的镜像
docker pull  redis
运行容器
docker run --name redis -p 6379:6379  -v /home/xiaobiao/docker-data/redis:/data -d redis redis-server --appendonly yes
-p 6379:6379 : 将容器的6379端口映射到主机的6379端口
-v /home/xiaobiao/docker-data/redis:/data   将主机中/home/xiaobiao/docker-data/redis挂载到容器的/data
redis-server --appendonly yes : 在容器执行redis-server启动命令，并打开redis持久化配置



查看容器启动情况
docker ps
启动容器
docker start d3fa109879f0
连接、查看容器
docker exec -it 43f7a65ec7f8 redis-cli
进入容器目录
docker exec -it d3fa109879f0 /bin/bash
docker 删除容器  
docker rm id
 ctrl+p ctrl+q 退出容器
看看你启动的时候有没有指定redis.conf。没有指定的话redis在内部自动维持一套配置。