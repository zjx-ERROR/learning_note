## 基本命令行操作概念
![概念](img/docker/1.png)

### pull
```
docker pull 镜像(:版本)
```


### run
```
docker run (-d#后台运行)(-p 80:80#端口映射) (--name 容器名) (-v 宿主目录:容器目录) (-d #后台运行) 镜像
  -v 数据卷名称:容器内部的路径 #docker会自动创建，会将容器内部自带的文件，存储在默认的存放路径中
  -v 路径:容器内部的路径 #这个路径下是空的
```

### ps
```
docker ps (-a #查看所有容器) (-q #返回正在运行容器id) (-qa #返回所有容器的id)#查看正在运行的容器
```

### exec
```
docker exec -it 容器 bash #进入容器执行命令
```

### rm
```
docker rm -f (容器)|($(docker ps -qa)#删除所有容器)
```

### rmi
```
docker rmi 镜像
```

### commit
```
docker commit 容器 镜像
```

### images
```
docker images
```

### build
```
docker build -t 镜像名称:[tag] .
```

### inspect
```
docker docker inspect #查看容器内部细节
```

### load
```
docker load -i 镜像文件名 #加载镜像文件
```

### start
```
docker start 容器 #启动容器
```

### restart
```
docker restart 容器 #重启容器
```

### stop
```
docker stop 容器 #正常停止容器运行
```

### kill
```
docker kill 容器 #强制停止容器运行
```

### top
```
docker top 容器 #查看容器内的进程
```

### logs
```
docker logs (-t #加入时间戳) (-f #跟随最新的日志打印) （--tail 数字 #显示最后多少条） 容器
```

### cp
```
docker cp 文件|目录 容器:容器内路径 #将宿主机复制到容器内部
docker cp 容器:容器内资源路径 目录 #将容器内资源拷贝到宿主机
```

### volume
```

docker volume create 数据卷名称 #创建数据卷，默认存放在/var/lib/docker/volumes/
docker volume inspect 数据卷名称 #查看数据卷信息
docker volume ls #查看全部数据卷
docker volume rm 数据卷名称 #删除数据卷
```

## dockerfile基本格式

### FROM
```
FROM 镜像(:版本)
```

### WORKDIR
```
WORKDIR 路径 #自动创建运行路径
```

### COPY
```
COPY 宿主机目录 容器目录 #宿主机文件拷贝到容器中
```

### RUN
```
RUN shell #运行shell语句
```

### CMD
```
CMD shell #在workdir下执行，cmd可以有多个，只以最后一个为准
```

### ADD

### EXPOSE

### VOLUME

### ENV

### ARG

### LABEL
 
### ONBUILD

### SHELL

## docker-compose
### yml文件以key: value方式来指定配置信息，多个信息以换行+缩进的方式来区分，在docker-compose.yml文件中，不要使用制表符

```
version: '版本号'
services: 
    服务名称:
        restart: always  #代表只要docker启动，那么这个容器就跟着启动
        image: 镜像名称 #指定镜像路径
        container_name: 容器名称  #指定容器名称
        ports: 
            - 3306:3306
            - 8080:8080 #指定端口号的映射
        enviroment:
            key: value #指定环境变量
        volumes:
            - 宿主机路径: 容器内路径 #映射数据卷
```

### docker-compose命令

#### docker-compose up -d #启动管理的容器
#### docker-compose down #关闭并删除容器
#### docker-compose start|stop|restart #开启|关闭|重启已经存在的由docker-compose维护的容器
#### docker-compose ps #查看由docker-compose维护的容器
#### docker-compose logs -f #查看日志

### docker-compose配置Dockerfile使用
```
version: '版本号'
services: 
    服务名称:
        restart: always  #代表只要docker启动，那么这个容器就跟着启动
        build: #构建自定义镜像
            context: ../ #指定Dockerfile文件所在路径
            dockerfile: Dockerfile #指定Dockerfile文件名称
        image: 镜像名称 #指定镜像路径
        container_name: 容器名称  #指定容器名称
        ports: 
            - 3306:3306
            - 8080:8080 #指定端口号的映射
        enviroment:
            key: value #指定环境变量
        volumes:
            - 宿主机路径: 容器内路径 #映射数据卷
```

#### docker-compose build #重新构建镜像
#### docker-compose up -d --build #运行前，重新构建


## 如何从容器获取宿主机ip
通过虚拟网桥ip用ssh链接宿主机，执行ip route get命令堆docker宿主机ip进行获取，附python代码
```python
import paramiko
s = paramiko.SSHClinet()
s.load_system_host_keys()
ssh_name = "root"
ssh_password = "root"
ssh_port = 22
docker_ip = "172.17.0.1"
s.set_missing_host_key_policy(paramiko.AutoAddPolicy())
s.connect(docker_ip,ssh_port,ssh_name,ssh_password,timeout=5)
stdin,stdout,stderr = s.exec_command("ip route get 8.8.8.8|awk '{print $7}'")
ip = stdout.read().decode("utf-8")
print("宿主机ip：",ip)
```
