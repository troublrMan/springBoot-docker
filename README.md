spring boot 应用发布到 docker 完整版

一、概述

    spring boot 和 docker 本身就不多介绍了，本文主要介绍使用 docker-maven-plugin 插件，直接将 spring boot 应用一键发布到 docker 容器中。
    文末会提供源码 Git 地址。
    笔者 docker 部署于一台 Centos 7.2 的云服务器，换做 VM 虚拟机的 Linux 也是一样的。
    用到的所涉及的软件版本皆为当前最新的，构建工具为 maven，如果使用的其他工具，请使用对应步骤替换。
 
二、安装并配置 Docker

    笔者用于测试的 linux 为 Centos，其他系统也差不太多。
    1、安装 Docker            	
    直接使用 yum 安装即可: sudo yum install docker
    安装完成后可以通过如下命令查看是否安装成功：docker version
    如果正常输出版本等相关信息，即表示安装成功。
    2、配置 Docker Remote API       		
    docker-maven-plugin 插件是使用的 Docker Remote API 进行远程提交镜像的，docker 默认并没有开启该选项，直接修改 docker 服务配置即可，
Centos 7 配置文件位于：/usr/lib/systemd/system/docker.service
    直接在 ExecStart 启动参数的 /usr/bin/dockerd 后面添加以开启 TCP 连接：-H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock
    也可在此增加 Docker Hub 镜像加速地址，修改完成后完整的配置如下：
[plain] view plain copy print?
[Unit]  
Description=Docker Application Container Engine  
Documentation=http://docs.docker.com  
After=network.target  
Wants=docker-storage-setup.service  
Requires=docker-cleanup.timer  
  
[Service]  
Type=notify  
NotifyAccess=all  
KillMode=process  
EnvironmentFile=-/etc/sysconfig/docker  
EnvironmentFile=-/etc/sysconfig/docker-storage  
EnvironmentFile=-/etc/sysconfig/docker-network  
Environment=GOTRACEBACK=crash  
Environment=DOCKER_HTTP_HOST_COMPAT=1  
Environment=PATH=/usr/libexec/docker:/usr/bin:/usr/sbin  
ExecStart=/usr/bin/dockerd --registry-mirror=https://xxoo.mirror.acs.aliyun.com -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock \  
          --add-runtime docker-runc=/usr/libexec/docker/docker-runc-current \  
          --default-runtime=docker-runc \  
          --exec-opt native.cgroupdriver=systemd \  
          --userland-proxy-path=/usr/libexec/docker/docker-proxy-current \  
          $OPTIONS \  
          $DOCKER_STORAGE_OPTIONS \  
          $DOCKER_NETWORK_OPTIONS \  
          $ADD_REGISTRY \  
          $BLOCK_REGISTRY \  
          $INSECURE_REGISTRY  
ExecReload=/bin/kill -s HUP $MAINPID  
LimitNOFILE=1048576  
LimitNPROC=1048576  
LimitCORE=infinity  
TimeoutStartSec=0  
Restart=on-abnormal  
MountFlags=slave  
  
[Install]  
WantedBy=multi-user.target  
    重新载入 systemd，扫描新的或有变动的单元(docker)：
systemctl daemon-reload docker
    启动 docker(如果已启动，则使用restart)：
systemctl start docker
    本地测试：
 curl http://localhost:2375
    如果没报错基本就是完成了。我的工作环境是windows的，没安装 curl 工具，直接在浏览器输入 http://ip:2375 也是可以远程测试的，如果无法访问，请检查 docker 所在服务器的防火墙配置等。
    3、下载父镜像
    这一步不是必须的，Docker 会根据 Dockerfile 生成镜像时自动下载需要的依赖，但是为了后面完成 Spring boot 应用后构建并发布到 Docker 时不至于傻傻的等，这里可以先将父镜像 pull 到本地。
    java8 运行环境的 docker 镜像目前最受欢迎的是 frolvlad/alpine-oraclejdk8 ，那我们就用他吧：
docker pull frolvlad/alpine-oraclejdk8
    这可能会耗时一会儿，不过可以放这儿不用管，直接进行下一步。
    通常不指定镜像版本，默认会 pull 当期镜像的 latest 版本。
 
三、完成一个 Spring boot 应用并添加 docker-maven-plugin 插件

    1、完成 Spring boot 应用
		
    Eclipse 可以安装 STS 插件，Idea 也自带 Spring Boot 工程一键生成工具，这里直接生成一个最简单的 Web 工程，并在启动代码中加入少许代码以对外提供一个简单的接口用于测试：
package com.anxpp.demo;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
@SpringBootApplication
@RestController
public class DockerApplication {
    public static void main(String[] args) {
        SpringApplication.run(DockerApplication.class, args);
    }
    @RequestMapping
    public String index(){
        return "Hello world !";
    }
}
    完成后可以运行并访问 http://localhost:8080 测试是否正常工作。
    2、添加 docker-maven-plugin 插件
		
    插件最新版本为 0.4.13，此处还是贴完整的 pom 文件吧：
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.anxpp.demo</groupId>
    <artifactId>docker</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>
    <name>docker</name>
    <description>Demo project for Spring Boot</description>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.4.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <properties>
        <docker.image.prefix>anxpp</docker.image.prefix>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <plugin>
                <groupId>com.spotify</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <version>0.4.13</version>
                <configuration>
                    <imageName>${docker.image.prefix}/${project.artifactId}</imageName>
                    <dockerHost>http://***.***.anxpp.com:2375</dockerHost>
                    <dockerDirectory>src/main/docker</dockerDirectory>
                    <resources>
                        <resource>
                            <targetPath>/</targetPath>
                            <directory>${project.build.directory}</directory>
                            <include>${project.build.finalName}.jar</include>
                        </resource>
                    </resources>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
    imageName 指定生成的镜像名称，dockerHost 指定 Docker Remote API 的远程访问地址，这里请自行按照实际情况填写。
    3、编写 Dockerfile
    我们需要在 src/main/docker 下添加 Dockerfile 文件(当然，这在上一步添加插件的时候也是可以配置的)，Dockerfile 如何写请参考 docker 相关文章，文末也会给连接。完整的文件内容如下：
FROM frolvlad/alpine-oraclejdk8
VOLUME /tmp
ADD docker-0.0.1-SNAPSHOT.jar app.jar
RUN sh -c 'touch /app.jar'
ENV JAVA_OPTS=""
ENTRYPOINT [ "sh", "-c", "java $JAVA_OPTS -Djava.security.egd=file:/dev/./urandom -jar /app.jar" ]
    ADD 后面请按照实际情况填写自己 package 生成的 jar 文件名称。
 
四、构建

    1、生成镜像
		
    首先需要运行 maven 的 package 命令生成 jar 文件，直接运行 docker:build 会因为找不到 jar 文件而构建失败！
mvn clean package
    现在的 IDE 也都会提供图形化工具，拖着鼠标点点点就行了。
    现在回头看看 二、3 中的父镜像有没有下载好，如果已经下载好即可进行下面的步骤（可以输入命令查看镜像列表）。
    打包完成后运行 docker:build 。
    如果一切顺利，那么镜像就生成好了并推送到 docker 中，输入如下命令查看镜像列表：
docker images
    如果列表中有我们刚刚生成的镜像，那么表示以上步骤都是成功的（废话）。
    2、运行
		
    镜像运行后就成为容器，使用如下命令运行启动刚刚生成的镜像：
docker run -d -p 8080:8080 -t anxpp/docker
    -d 是让容器后台运行
    -p 是将容器内的端口映射到 docker 所在系统的端口
    -t 是打开一个伪终端，以便后续可以进入查看控制台 log
    查看运行中的容器：
docker ps
    可以使用浏览器访问测试是否一切都很顺利：
    http://***.***.anxpp.com:8080
    那么，用 docker 构建、运行和发布一个 Spring Boot 应用到此源码结束，深藏功与名。
 
五、多说几句

    Docker 文章参考：Docker基础教程及实践专栏
    Spring boot 应用源码 GitHub：https://github.com/anxpp/springboot-docker-demo.git
    有问题赶紧留言吧，趁我还记得...
