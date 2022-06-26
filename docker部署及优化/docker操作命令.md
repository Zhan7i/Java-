## docker操作命令

**修改环境变量**

```bash
vim /etc/profile
输入：    export PATH=$PATH:/...  # PATH后添加绝对路径
source /etc/profile  # 使修改生效
```

**重启docker** 	

```
systemctl restart docker
```

**启动所有容器** 	

```
docker start $(docker ps -a | awk '{ print $1}' | tail -n +2)
```

**启动mysql容器**	

```
docker run --name mysql -e MYSQL_ROOT_PASSWORD=rood -d mysql:8.0.20
```

```

docker run \
-p 3306:3306 \
--name mysql \
--privileged=true \
--restart unless-stopped \
-v /home/admin/HC/mysql:/etc/mysql \
-v /home/admin/HC/mysql/log:/logs \
-v /home/admin/HC/mysql/data:/var/lib/mysql \
-v /etc/localtime:/etc/localtime \
-e MYSQL_ROOT_PASSWORD=root \
-d mysql:8.0.20
```

**启动redis容器**	

```
docker run -itd --name redis -p 6379:6379 redis
```

**启动kafka容器** 

```
docker run --name kafka \
-p 9092:9092 \
-e KAFKA_BROKER_ID=0 \
-e KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181 \
-e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092 \
-e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092 \
-d wurstmeister/kafka:2.12-2.0.1
```

**创建Portainer docker容器管理工具**

```
docker run -d -p 9000:9000 --restart=always -v /var/run/docker.sock:/var/run/docker.sock --name portainer  docker.io/portainer/portainer
```

**创建镜像** 

```
docker build  -t      image-name   ./imagefilepath 
```

**启动zookeeper容器**	

```
docker run -d --name zookeeper -p 2181:2181 -t wurstmeister/zookeeper
```

**删除docker部署的应用**      

```
  docker rmi e123456123(服务启动的ID号)
```

**删除docker部署的container** 	

```
docker rm e12314546(container的ID号)
```

**删除所有镜像**	

```
docker rmi $(docker images -q)
```

**删除所有容器** 	

```
docker rm $(docker ps -a -q)
```

**杀死某一端口**	

```
fuser -kill xxx(端口号)/tcp
```

**查看服务的id号**  

```
docker images
```

**查看正在运行的容器**	

```
docker ps 
```

**创建网段**  

```
docker network create--subnet=192.168.50.0/24  newnetworkname
```

**查看所有的容器（包括未运行）**	

```
docker ps -a
```

**查看容器ip**

```
 docker inspect  name
```

**创建私服**	

```
docker pull registry   docker run -d -p 5000:5000 --restart=always --name registry  registry
```

- ```java
  docker run -d \
  -p 5000:5000 \
  --restart=always \
  --name registry \
  -v /home/admin/HC/docker_registry/docker_image_repo:/var/lib/registry \
  -e REGISTRY_STORAGE_DELETE_ENABLED=true \
  --privileged=true \
  registry
  
  #参数解释
  -e REGISTRY_STORAGE_DELETE_ENABLED=true:registry私有仓库,允许删除
      
      
  docker run -it \
  -p 8080:80 \
  --name registry-frontend \
  --restart=always \
  -v /home/admin/HC/docker_registry/certs/frontend.crt:/etc/apache2/server.crt:ro \
  -v /home/admin/HC/docker_registry/certs/frontend.key:/etc/apache2/server.key:ro \
  -v /home/admin/HC/docker_registry/docker_image_repo:/var/lib/registry \
  -e REGISTRY_READONLY=false \
  -e ENV_DOCKER_REGISTRY_HOST=192.168.48.128 \
  -e ENV_DOCKER_REGISTRY_PORT=5000 \
  konradkleine/docker-registry-frontend:v2
      
  #参数解释
  -e REGISTRY_READONLY=false :  registry不为只读,允许删除
  ```


**查看私服仓库**	

```
curl http://localhost:5000/v2/_catalog
```

**上传的镜像打tag**   

```
docker tag  localimages    192.168.48.128:5000/localimages:latest
```

**删除私服镜像**	

```
docker exec <容器名> rm -rf /var/lib/registry/docker/registry/v2/repositories/<镜像名>
```

**上传私服** 	

```
docker  push  192.168.48.128:5000/localimages:latest
```

- 可能遇到的问题：

- ```bash
  在/etc/docker/目录下，创建daemon.json文件。在文件中写入：
  
  {
      "insecure-registries": [
          "registry:5000"
      ]
  }
  #需要在/etc/hosts里映射registry对应的host
  ```

**修改乱码问题**	:引入centos中文版 

```java
FROM centos:Chinese

# 新建目录
RUN mkdir /usr/local/java
WORKDIR /usr/local/java
# 将jdk文件拷贝到容器/usr/local/java/并解压
ADD jdk-8u321-linux-x64.tar.gz /usr/local/java

# 软连接
RUN ln -s /usr/local/java/jdk1.8.0_181 /usr/local/java/jdk

# 设置环境变量
ENV JAVA_HOME /usr/local/java/jdk1.8.0_321
ENV JRE_HOME ${JAVA_HOME}/jre
ENV CLASSPATH .:${JAVA_HOME}/lib:${JRE_HOME}/lib
ENV PATH ${JAVA_HOME}/bin:$PATH
```

**docker 启动mysql权限问题** 	 

```
暂时性修改权限： (不需要重启机器) 
setenforce 0    然后 getenforce查看


永久性修改权限：
修改/etc/selinux/config 文件  (修改配置文件需要重启机器：)
将SELINUX=enforcing改为SELINUX=disabled
```

**docker.sock 权限问题** 

```
cd /var/run 

chmod 777 docker.sock
```

linux查看防火墙  	

```
systemctl status firewalld
```

linux关闭防火墙	

```
systemctl disable firewalld
```

linux查看端口8001被哪个进程占用	

```
netstat -ano | grep  "8001" 	  
ps -ef | grep 8001   
lsof -i:8001
```

linux杀死进程   

```
 kill -9  5464
```

windows查看端口8001被哪个进程占用	 

```
netstat -ano | findstr "8001"  
```

windows查看进程号为3736对应的进程	

```
tasklist | findstr "3736"
```

windows结束该进程	

```
taskkill /f /t /im java.exe
```

docker 删除none标签镜像:

```bash
docker rmi `docker images | grep  "<none>" | awk '{print $3}'`
如果确定所有none镜像确实没用，直接加个-f强制删除
docker rmi -f `docker images | grep  "<none>" | awk '{print $3}'`
```

