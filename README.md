
## 1.部署微服务集群

**需求**：将之前学习的cloud-demo微服务集群利用DockerCompose部署

**实现思路**：

① 查看课前资料提供的cloud-demo文件夹，里面已经编写好了docker-compose文件

② 修改自己的cloud-demo项目，将数据库、nacos地址都命名为docker-compose中的服务名

③ 使用maven打包工具，将项目中的每个微服务都打包为app.jar

④ 将打包好的app.jar拷贝到cloud-demo中的每一个对应的子目录中

⑤ 将cloud-demo上传至虚拟机，利用 docker-compose up -d 来部署



## 1.1.compose文件

查看课前资料提供的cloud-demo文件夹，里面已经编写好了docker-compose文件，而且每个微服务都准备了一个独立的目录：

![](https://img-blog.csdnimg.cn/img_convert/04a3aac444d22444929dbd5408b2ab85.png)

内容如下：

```yaml
version: "3.2"

services:
  nacos:
    image: nacos/nacos-server
    environment:
      MODE: standalone
    ports:
      - "8848:8848"
  mysql:
    image: mysql:5.7.25
    environment:
      MYSQL_ROOT_PASSWORD: 123
    volumes:
      - "$PWD/mysql/data:/var/lib/mysql"
      - "$PWD/mysql/conf:/etc/mysql/conf.d/"
  userservice:
    build: ./user-service
  orderservice:
    build: ./order-service
  gateway:
    build: ./gateway
    ports:
      - "10010:10010"
```

可以看到，其中包含5个service服务：

- `nacos`：作为注册中心和配置中心
  - `image: nacos/nacos-server`： 基于nacos/nacos-server镜像构建
  - `environment`：环境变量
    - `MODE: standalone`：单点模式启动
  - `ports`：端口映射，这里暴露了8848端口
- `mysql`：数据库
  - `image: mysql:5.7.25`：镜像版本是mysql:5.7.25
  - `environment`：环境变量
    - `MYSQL_ROOT_PASSWORD: 123`：设置数据库root账户的密码为123
  - `volumes`：数据卷挂载，这里挂载了mysql的data、conf目录，其中有我提前准备好的数据
- `userservice`、`orderservice`、`gateway`：都是基于Dockerfile临时构建的



查看mysql目录，可以看到其中已经准备好了cloud_order、cloud_user表：

![](https://img-blog.csdnimg.cn/img_convert/424c6a2acd528f6cb082d6c9fa05b872.png)

查看微服务目录，可以看到都包含Dockerfile文件：

![](https://img-blog.csdnimg.cn/img_convert/a5ffc33a3d3427f99d897988dfcdaa65.png)

内容如下：

```dockerfile
FROM java:8-alpine
COPY ./app.jar /tmp/app.jar
ENTRYPOINT java -jar /tmp/app.jar
```

## 1.2.修改微服务配置

因为微服务将来要部署为docker容器，而容器之间互联不是通过IP地址，而是通过容器名。这里我们将order-service、user-service、gateway服务的mysql、nacos地址都修改为基于容器名的访问。

如下所示：

```yaml
spring:
  datasource:
    url: jdbc:mysql://mysql:3306/cloud_order?useSSL=false
    username: root
    password: 123
    driver-class-name: com.mysql.jdbc.Driver
  application:
    name: orderservice
  cloud:
    nacos:
      server-addr: nacos:8848 # nacos服务地址
```



## 1.3.打包

接下来需要将我们的每个微服务都打包。因为之前查看到Dockerfile中的jar包名称都是app.jar，因此我们的每个微服务都需要用这个名称。

可以通过修改pom.xml中的打包名称来实现，每个微服务都需要修改：

```xml
<build>
  <!-- 服务打包的最终名称 -->
  <finalName>app</finalName>
  <plugins>
    <plugin>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-maven-plugin</artifactId>
    </plugin>
  </plugins>
</build>
```

打包后：

![](https://img-blog.csdnimg.cn/img_convert/ff4be4a77d00270cdcd4d25879567783.png)

## 1.4.拷贝jar包到部署目录

编译打包好的app.jar文件，需要放到Dockerfile的同级目录中。注意：每个微服务的app.jar放到与服务名称对应的目录，别搞错了。

user-service：

![](https://img-blog.csdnimg.cn/img_convert/02929c922a1f2636f81c7fdbb15dc3aa.png)

order-service：

![](https://img-blog.csdnimg.cn/img_convert/dde1c87abc5af0be2ab9c5ca9e324cd2.png)

gateway：

![](https://img-blog.csdnimg.cn/img_convert/c826f0b5ebb479166992ce8ee8dcb9bf.png)

## 1.5.部署

最后，我们需要将文件整个cloud-demo文件夹上传到虚拟机中，理由DockerCompose部署。

上传到任意目录：

![image-20230204150736895](https://img-blog.csdnimg.cn/img_convert/7c629a2a9d389fd95c6e400a77c98cd2.png)

部署：

进入cloud-demo目录，然后运行下面的命令：

```sh
docker-compose up -d
```

![image-20230204150748941](https://img-blog.csdnimg.cn/img_convert/ce336c64f6c4fcf354ee59193107a0f9.png)

