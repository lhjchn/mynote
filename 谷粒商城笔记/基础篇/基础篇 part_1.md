# 基础篇 part_1

#### **一、项目简介**

基于B2C模式的分布式商城项目

#### **二、分布式基础概念**

集群是个物理形态，分布式是个工作方式。

远程调用：在分布式系统中，各个服务可能处于不同主机，但是服务之间不可避免的相互调用，我们成为远程调用

springcloud中使用HTTP+JSON的方式完成远程调用。

##### **服务注册/发现&注册中心**

A服务调用B服务，A服务并不知道B服务当前在哪几台服务器有，哪些是正常的，哪些服务已经下线。解决这个问题可以引入注册中心。配置中心用来几种管理微服务的配置信息。

##### **服务熔断&服务降级**

​		在微服务架构中，微服务之间通过网络进行通信，存在相互依赖，当其中一个服务不可用时，有可能会造成雪崩效应。要防止这样的情况，必须要有容错机制来保护服务。

​		服务雪崩：rpc远程调用情景，订单服务 --> 商品服务 --> 库存服务，库存服务出现故障导致响应慢，导致商品服务需要等待，可能等到10s后库存服务才能响应。库存服务的不可用导致商品服务阻塞，商品服务等的期间，订单服务也处于阻塞。一个服务不可用导致整个服务链都阻塞。如果是高并发，第一个请求调用后阻塞10s得不到结果，第二个请求直接阻塞10s。更多的请求进来导致请求积压，全部阻塞，最终服务器的资源耗尽。

==解决方案：==

1. 服务熔断：指定超时时间，库存服务3s没有响应就超时，如果经常失败，比如10s内100个请求都失败了。开启断路保护机制，下一次请求进来不调用库存服务了，因为上一次100%错误都出现了，我们直接在此中断，商品服务直接返回，返回一些默认数据或者null，而不调用库存服务了，这样就不会导致请求积压。设置服务的超时，当被调用的服务经常失败到达某个阈值，我们可以开启断路保护机制，后来的请求不再去调用这个服务，本地直接返回默认的数据。
2. 服务降级：在运维期间，当系统处于高峰期，系统资源紧张，我们可以让非核心业务降级运行。降级：某些服务不处理，或者处理简单【抛异常、返回NULL、调用Mock数据、调用Fallback处理逻辑】。

##### **API网关**

​		客户端发送请求到服务器路途中，设置一个网关，请求都先到达网关，网关对请求进行统一认证（合法非法）和处理等操作。他是安检。在微服务架构中，API gateway作为整体架构的重要组件，它抽象了微服务中都需要的公共功能，同时提供了客户端负载均衡，服务自动熔断，灰度发布，统一认证，限流流控，日志统计等丰富的功能，帮助我们解决很多API管理难题。

##### **整体技术架构**

![b3a972ccdb5aa17ffaa4a443542032f0](https://gitee.com/lhjchn/picgo/raw/master/img/b3a972ccdb5aa17ffaa4a443542032f0.png)

​		前后分离开发，分为内网部署和外网部署，外网是面向公众访问的，部署前端项目，可以有手机APP，电脑网页；内网部署的是后端集群。  
​		前端在页面上操作发送请求到后端，在这途中会经过Nginx集群，Nginx把请求转交给API网关(<font color='red'>springcloud gateway</font>)（网关可以根据当前请求动态地路由到指定的服务，看当前请求是想调用商品服务还是购物车服务还是检索），从路由过来如果请求很多，可以负载均衡地调用商品服务器中一台（商品服务复制了多份），当商品服务器出现问题也可以在网关层面对服务进行熔断或降级（使用阿里的<font color='red'>Sentinel组件</font>），网关还有其他的功能如认证授权、限流（只放行部分到服务器）等。  
​		请求到达服务器后进行处理（springboot为微服务），服务与服务可能会相互调用（<font color='red'>使用OpenFeign组件</font>），有些请求可能经过登录才能访问（基于OAuth2.0的认证中心。安全和权限使用<font color='red'>springSecurity</font>控制）  
​		服务需要保存一些数据或者需要使用缓存，我们使用<font color='red'>redis集群（分片+哨兵集群）</font>，持久化使用mysql，读写分离和分库分表。  
​		服务和服务之间会使用<font color='red'>消息队列（RabbitMQ）</font>，来完成异步解耦，分布式事务的一致性。  
​		有些服务可能需要全文检索，检索商品信息，使用<font color='red'>ElaticSearch</font>。  
​		服务可能需要存取数据，使用阿里云的<font color='red'>对象存储服务OSS</font>。  
​		项目上线后为了快速定位问题，使用<font color='red'>ELK</font>对日志进行处理，使用LogStash收集业务里的各种日志，把日志存储到ES中，用Kibana可视化页面从ES中检索出相关信息，帮助我们快速定位问题所在。  
​		在分布式系统中，由于我们每个服务都可能部署在很多台机器，服务和服务可能相互调用，就得知道彼此都在哪里，所以需要将所有服务都注册到注册中心。服务从注册中心发现其他服务所在位置（使用<font color='red'>阿里Nacos作为注册中心</font>）。  
​		每个服务的配置众多，为了实现改一处配置相同配置就同步更改，使用阿里的<font color='red'>Nacos做配置中心</font>，服务从配置中心中动态取配置。  		服务追踪，追踪服务调用链哪里出现问题，使用springcloud提供的<font color='red'>Sleuth、Zipkin、Metrics</font>，把每个服务的信息交给开源的<font color='red'>Prometheus</font>进行聚合分析，再由<font color='red'>Grafana</font>进行可视化展示，提供Prometheus提供的<font color='red'>AlterManager</font>实时得到服务的告警信息，以短信/邮件的方式告知服务开发人员。  
​		还提供了持续集成和持续部署。项目发布起来后，因为微服务众多，每一个都打包部署到服务器太麻烦，有了持续集成后开发人员可以将修改后的代码提交到github，运维人员可以通过自动化工具<font color='red'>Jenkins Pipeline</font>将github中获取的代码打包成docker镜像，最终是由<font color='red'>k8s</font>集成docker服务，将服务以docker容器的方式运行。

##### **微服务划分图**

![img](https://gitee.com/lhjchn/picgo/raw/master/img/143e5f57108634fea45ffc9f20f5963f.png)

前端项目分为admin-vue（工作人员使用的后台管理系统）、shop-vue（面向公众访问的web网站）

后端项目：

gulimall  
├── gulimall-common -- 工具类及通用代码  
├── renren-generator -- 人人开源项目的代码生成器  
├── gulimall-auth-server -- 认证中心（社交登录、OAuth2.0、单点登录）  
├── gulimall-cart -- 购物车服务  
├── gulimall-coupon -- 优惠卷服务  
├── gulimall-gateway -- 统一配置网关  
├── gulimall-order -- 订单服务：订单增删改查、验价、幂等性token  
├── gulimall-product -- 商品服务：商品的增删改查、商品的上下架、商品详情  
├── gulimall-search -- 检索服务：商品的检索ES  
├── gulimall-seckill -- 秒杀服务：定时任任务、redis  
├── gulimall-third-party -- 第三方服务  
├── gulimall-ware -- 仓储服务：商品的库存  
├── renrenfast -- 后台管理服务  
└── gulimall-member -- 会员服务：用户的个人中心、收货地址

#### **三、linux环境搭建**

##### 1.快速安装虚拟机

1. VisualBox进行安装需要cpu开启虚拟化，在BIOS设置主板，CPU configuration，然后点击Intel Vitualization Technology，VitualBox + Vagrant可以帮助我们快速创建出一个虚拟机。

2. Vagrant有对应的镜像仓库，地址为https://app.vagrantup.com/boxes/search，cmd命令行输入`vagrant init centos/7`，即可初始化一个centos7系统。（注意这个命令在哪个目录下执行的，他的Vagrantfile就生成在哪里），输入`vagrant up` 启动虚拟机环境

   ==vagrant up执行报错：==process_builder.rb:43:in `join': incompatible character encodings: GBK and UTF-8 (Encoding::CompatibilityError) 解决方法：vagrant init 一个centos系统时，命令在无中文的目录下执行，并且一个系统对应一个Vagrantfile

3. 通过输入`vagrant ssh` 就会连上虚拟机

4. 默认vagrant初始化的虚拟机网络方式是==网络地址转换NAT==（端口转发），如果其他主机要访问虚拟机，必须由windows端口如3333转发给虚拟机端口如3306。这样每在linux里安一个软件都要进行端口映射，不方便，（也可以在virualBox里挨个设置）。我们想要给虚拟机一个固定的ip地址，关闭防火墙，windows和虚拟机可以互相ping通。

   - 方式1：是在虚拟机中配置静态ip，具体操作：（VirualBox中第二个网卡eth1中设置）


   ![image-20220117174250122](https://gitee.com/lhjchn/picgo/raw/master/img/image-20220117174250122.png)

   ```shell
   #修改：
   BOOTPROTO=static
   ONBOOT=yes
   #添加：
   IPADDR=192.168.56.10
   NETMASK=255.255.255.0
   GATEWAY=192.168.56.2
   DNS1=114.114.114.114
   #重启网络
   service network restart/systemctl restart network
   ```

   - 方式2：更改Vagrantfile文件中的虚拟机ip，修改其中的config.vm.network "private_network",ip:"<font color='red'>192.168.56.10</font>"，这个ip需要在windows的ipconfig中查到vitualbox的虚拟网卡ip，然后使用同网段的一个其他ip（不能是1，1是我们的主机）。配置完后`vagrant reload` 重启虚拟机。在虚拟机中ip addr就可以查看到地址了，互相ping也能ping通。

5. 默认只允许ssh登录方式，为了后来操作方便，文件上传等，我们可以配置使用账号密码登录（<font color='red'>默认账号 密码root/vagrant</font>）

   ```shell
   vim /etc/ssh/sshd_config
   #修改
   PasswordAuthentication yes
   #重启
   service sshd restart
   #通过客户端工具连接192.168.56.10:22端口成功，用户名root，密码vagrant
   ```

6. 修改yum源，提高下载速度

   ```shell
   #备份原yum源
   mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
   #使用新yum源，该命令会下载163的yum源，并重命名
   curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.163.com/.help/CentOS7-Base-163.repo
   # 生成缓存
   yum makecache
   ```

##### **2.虚拟机安装docker**

https://docs.docker.com/engine/install/centos/

```shell
#卸载系统之前的docker 
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
                  
                  
sudo yum install -y yum-utils

# 配置镜像
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
    
sudo yum install docker-ce docker-ce-cli containerd.io

sudo systemctl start docker
# 设置开机自启动
sudo systemctl enable docker

docker -v
sudo docker images

# 配置镜像加速 https://cr.console.aliyun.com/cn-qingdao/instances/mirrors
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://chqac97z.mirror.aliyuncs.com"]
}
EOF

sudo systemctl daemon-reload
sudo systemctl restart docker
```

##### 3.安装mysql-docker

```shell
#从镜像仓库拉取mysql镜像
sudo docker pull mysql:5.7

# --name指定容器名字 -v目录挂载 -p指定端口映射  -e设置mysql参数 -d后台运行
sudo docker run -p 3306:3306 --name mysql \
-v /mydata/mysql/log:/var/log/mysql \
-v /mydata/mysql/data:/var/lib/mysql \
-v /mydata/mysql/conf:/etc/mysql \
-e MYSQL_ROOT_PASSWORD=root \
-d mysql:5.7

# 进入已启动的容器
docker exec -it mysql bin/bash
# 退出进入的容器
exit;

#修改mysql的编码
#因为有目录映射，所以我们可以直接在镜像外的宿主机上执行
vi /mydata/mysql/conf/my.conf 
#保存，注意不是collection而是collation)
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8
[mysqld]
init_connect='SET collation_connection = utf8_unicode_ci'
init_connect='SET NAMES utf8'
character-set-server=utf8
collation-server=utf8_unicode_ci
skip-character-set-client-handshake
skip-name-resolve

docker restart mysql

# 设置mysql跟随docker一起启动
docker update mysql --restart=always
```

##### 4.安装redis-docker

==如果直接挂载的话docker会以为挂载的是一个目录==，所以我们先创建一个文件然后再挂载，在虚拟机中。

```shell
# 创建目录和文件
mkdir -p /mydata/redis/conf
touch /mydata/redis/conf/redis.conf

docker pull redis

docker run -p 6379:6379 --name redis \
-v /mydata/redis/data:/data \
-v /mydata/redis/conf/redis.conf:/etc/redis/redis.conf \
-d redis redis-server /etc/redis/redis.conf

# 直接进去redis客户端
docker exec -it redis redis-cli

#设置redis容器在docker启动的时候启动
docker update redis --restart=always
```

默认是不持久化的。在redis.conf 配置文件中输入appendonly yes，就可以aof持久化了，修改完docker restart redis，docker -it redis redis-cli

##### 5.安装nginx docker

```shell

docker pull nginx
# 随便启动一个nginx实例，只是为了复制出配置，放到docker里作为镜像的统一配置
docker run -p 80:80 --name nginx -d nginx:1.10

# 把nginx里的/etc/nginx文件夹复制出来
cd /mydata/nginx
docker cp nginx:/etc/nginx .
# 将文件复制到conf中
mv /mydata/nginx/nginx/* /mydata/nginx/conf
# 停掉nginx
docker stop nginx
docker rm nginx

# 创建新的nginx，使用刚才复制出来的配置文件
docker run -p 80:80 --name nginx \
-v /mydata/nginx/html:/usr/share/nginx/html \
-v /mydata/nginx/logs:/var/log/nginx \
-v /mydata/nginx/conf:/etc/nginx \
-d nginx:1.10

# 注意一下这个路径映射到了/usr/share/nginx/html，我们在nginx配置文件中是写/usr/share/nginx/html，不是写/mydata/nginx/html

docker update nginx --restart=always

#测试 浏览器访问：http://192.168.56.10
cd /mydata/nginx/html/
vim index.html
```

#### **四、Git的安装配置**

```shell
# 配置用户名，通过该用户进行识别
git config --global user.name "username" //(名字，随意写)

# 配置邮箱
git config --global user.email "55333@qq.com" // 注册账号时使用的邮箱

# 配置ssh免密登录
ssh-keygen -t rsa -C "55333@qq.com"
三次回车后生成了密钥：公钥私钥
cat ~/.ssh/id_rsa.pub

也可以查看密钥
浏览器登录码云后，个人头像上点设置--ssh公钥---随便填个标题---复制
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC6MWhGXSKdRxr1mGPZysDrcwABMTrxc8Va2IWZyIMMRHH9Qn/
wy3PN2I9144UUqg65W0CDE/thxbOdn78MygFFsIG4j0wdT9sdjmSfzQikLHFsJ02yr58V6J2zwXcW9AhIlaGr+X
IlGKDUy5mXb4OF+6UMXM6HKF7rY9FYh9wL6bun9f1jV4Ydlxftb/xtV8oQXXNJbI6OoqkogPKBYcNdWzMbjJdmbq
2bSQugGaPVnHEqAD74Qgkw1G7SIDTXnY55gBlFPVzjLWUu74OWFCx4pFHH6LRZOCLlMaJ9haTwT2DB/sFzOG/Js+c
EExx/arJ2rvvdmTMwlv/T+6xhrMS3 553736044@qq.com

# 测试
ssh -T git@gitee.com
测试成功，就可以无密给码云推送仓库了
```



#### 五、项目搭建

1. 在码云新建仓库，仓库名gulimall，选择语言java，在.gitignore选中maven（就会忽略掉maven一些个人无需上传的配置文件），许可证选Apache-2.0，开发模型选生成/开发模型，开发时在dev分支，发布时在master分支，创建。

2. 在IDEA中New–Project from version control–git–复制刚才项目的地址

3. IDEA然后New Module–Spring Initializer–com.atguigu.gulimall ， Artifact填 gulimall-product。Next—选择web（web开发），springcloud routing里选中openFeign（rpc调用）

4. 依次创建出以下服务模块

   - 商品服务product

   - 存储服务ware
   - 订单服务order
   - 优惠券服务coupon
   - 用户服务member

   共同点：

   - 导入web和openFeign

   - group：com.atguigu.gulimall

   - Artifact：gulimall-XXX

   - 每一个服务，包名com.atguigu.gulimall.XXX{product/order/ware/coupon/member}

   - 模块名：gulimall-XXX

     

   父工程中加入pom.xml文件，用于聚合服务，可以进行统一的编译打包等操作

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
   	<modelVersion>4.0.0</modelVersion>
   
   	<groupId>com.atguigu.gulimall</groupId>
   	<artifactId>gulimall</artifactId>
   	<version>0.0.1-SNAPSHOT</version>
   	<name>gulimall</name>
   	<description>聚合服务</description>
   
   	<packaging>pom</packaging>
   
   	<modules>
   		<module>gulimall-coupon</module>
   		<module>gulimall-member</module>
   		<module>gulimall-order</module>
   		<module>gulimall-product</module>
   		<module>gulimall-ware</module>
   	</modules>
   
   </project>
   ```

在maven窗口刷新，并点击+号，找到刚才的pom.xml添加进来，发现多了个root。这样比如运行root的clean命令，其他项目也一起clean了

修改总项目的`.gitignore`，把小项目里的垃圾文件在提交的时候忽略掉，<font color="red">注意需要提交之前就进行忽略</font>。在version control-->>local Changes，点击刷新看Unversioned Files，可以看到变化。选中需要进行管理的文件，选择右键、Add to VCS

```yaml
**/mvnw
**/mvnw.cmd
**/.mvn
**/target/
.idea
**/.gitignore
```

==具体可以参考：==https://www.cnblogs.com/youyoui/p/8337147.html

#### **六、数据库**

​		所有的数据库数据再复杂也不建立外键，因为在电商系统里，数据量大，做外键关联很耗性能。  
​		安装powerDesigner软件，其中name是给我们看的，code才是数据库里真正的信息。选择primary和identity作为主键。然后点preview就可以看到生成这张表的语句。点击菜单栏database–generate database—点击确定。  
​		使用连接sql的客户端执行我们的操作，建立数据库，字符集选utf8mb4，他能兼容utf8且能解决一些乱码的问题。导入sql，创建表<font color="red">（注意sql文件里没有建库语句）</font>。





