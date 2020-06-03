# docker-registry

网址：[yun.nju.edu.cn:10080](http://yun.nju.edu.cn:10080/home)

## 搭建registry

```bash
sudo service docker start  //启动docker
sudo docker pull registry  //下载registry镜像

mkdir -p /opt/data/registry  //创建Repositories存储目录

# 启动 private_registry
# -d : 让容器可以后台运行
# -p ：指定映射端口（前者是宿主机的端口号，后者是容器的端口号）
# -v ：数据挂载（前者是宿主机的目录，后者是容器的目录）
# --name : 为运行的容器命名
sudo docker run -d -p 5000:5000 -v /opt/data/registry:/var/lib/registry --name private_registry registry 

# 修改 /etc/docker/daemon, 加入 "insecure-registries": ["192.168.1.106:5000"]

sudo service docker restart   //重启容器
sudo docker start private_registry   //重启registry服务
```

## 搭建registry web

```bash
# ENV_DOCKER_REGISTRY_HOST 为 搭建registry服务器ip
# ENV_DOCKER_REGISTRY_PORT 为 搭建registry服务器端口
# -p 10080:80，10080为启动registry容器的服务器转发端口，80为registry web容器端口

sudo docker run \
  -d \
  -e ENV_DOCKER_REGISTRY_HOST=192.168.1.106 \
  -e ENV_DOCKER_REGISTRY_PORT=5000 \
  -p 10080:80 \
  --name registry_web \
  konradkleine/docker-registry-frontend:v2
```

## 如何管理private registry

### 重启

如果遇到断电、重启等因素导致private registry容器退出，应执行以下命令重新打开服务：

```bash
# 1. 重新打开 private registry
docker start private_registry
# 2. 尝试重新打开 registry web
docker start registry_web
# 3. 确认registry web已经打开，查看registry web容器的状态，若已运行则执行第4步，否则跳过
docker ps -a 
# 4. 重新搭建registry web容器
docker rmi registry_web

sudo docker run \
  -d \
  -e ENV_DOCKER_REGISTRY_HOST=192.168.1.106 \
  -e ENV_DOCKER_REGISTRY_PORT=5000 \
  -p 10080:80 \
  --name registry_web \
  konradkleine/docker-registry-frontend:v2
```

## 删除仓库中镜像

首先安装delete-docker-registry-image脚本，参考  https://github.com/burnettk/delete-docker-registry-image 

```bash
# 调用删除镜像脚本删除镜像，注：不用加private registry地址
delete_docker_registry_image --image tensorflow:tensorflow-py36-cu101
# 清理镜像
docker exec -it private_registry bin/registry garbage-collect /etc/docker/registry/config.yml
```

## 如何使用private registry

### 修改/etc/docker/daemon.json

```
# 加入 "insecure-registries": ["yun.nju.edu.cn:5000"]
vim /etc/docker/daemon.json
# 修改完成之后重启
service  docker restart
```

### 拉取一个镜像

```
sudo docker pull pytorch/pytorch:1.2-cuda10.0-cudnn7-runtime
```

### 修改镜像

```bash
## run
docker run --runtime=nvidia -it pytorch/pytorch:1.2-cuda10.0-cudnn7-runtime bash

## modify image

## ctrl + p + q 退出容器

## commit
docker commit <container id> <imagename>
```



### 使用tag指令将镜像指向到私有仓库

```bash
sudo docker tag pytorch/pytorch:1.2-cuda10.0-cudnn7-runtime 192.168.1.106:5000/pytorch:1.2-cuda10.0-cudnn7-runtime
```

###  将镜像推送到私有仓库

```bash
sudo docker push 192.168.1.106:5000/pytorch:1.2-cuda10.0-cudnn7-runtime
```

### 完成

此时在  http://yun.nju.edu.cn:10080/home  已经可以查看到刚刚上传的镜像