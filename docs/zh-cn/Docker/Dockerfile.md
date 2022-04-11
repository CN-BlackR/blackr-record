# Dockerfile

>
> [本文列举了部分常用的配置，完整版跳转到官方文档](https://docs.docker.com/engine/reference/builder/#from)

```Dockerfile
# 不要将根目录/用作PATH构建上下文，因为它会导致构建将硬盘驱动器的全部内容传输到 Docker 守护程序。
# https://docs.docker.com/engine/reference/builder/#from

# 声明参数，在其他地方使用：${arg}
ARG JDK_VERSION=17.0.2
ARG JAR_FILE=target/springbootApplication.jar

# FROM [--platform=<platform>] <image>[@<digest>] [AS <name>]
# --platform：linux/amd64， linux/arm64、 或windows/amd64。不写默认当前系统平台-platform=$BUILDPLATFORM
# image：镜像名称
# digest：tag、版本号，不写默认latest
# name：别名
FROM --platform=linux/amd64 openjdk:${JDK_VERSION} AS jdk17

# RUN，两种方式
# RUN <command>（shell形式，命令在 shell 中运行，在 Linux：默认/bin/sh -c 或 Windows 上：cmd /S /C）
# RUN ["executable", "param1", "param2"]（执行形式）
# RUN在下一次构建期间，指令缓存不会自动失效。可以通过使用--no-cache 标志来失效，例如docker build --no-cache
# build 过程中要运行的命令。
RUN mkdir -p /app_test_demo

# EXPOSE
# EXPOSE <port> [<port>/<protocol>...]
# 通知 Docker 容器在运行时侦听指定的网络端口。可以指定端口监听 TCP 还是 UDP，如果不指定协议，则默认为 TCP。
EXPOSE 80/udp
EXPOSE 80/tcp

# ENV，环境变量
ENV TZ=Asia/Shanghai JAVA_OPTS="-Xms128m -Xmx256m -Djava.security.egd=file:/dev/./urandom"

# ADD，两种形式
# ADD [--chown=<user>:<group>] <src>... <dest>
# ADD [--chown=<user>:<group>] ["<src>",... "<dest>"]    可包含空格的路径。
# <user> ：linux用户
# <group>：linux组
# --chown不适用于 Windows 容器
ADD --chown=55:mygroup files* /somedir/
# 从<src>路径复制新文件、目录或远程文件URL，并将它们添加到镜像的文件系统中<dest>
ADD hom* /mydir/
ADD hom?.txt /mydir/
# <dest>是绝对路径，或相对于WORKDIR的路径，WORKDIR源将在目标容器内复制到其中。
# ADD test.txt relativeDir/  ==  ADD test.txt <WORKDIR>/relativeDir/
ADD test.txt relativeDir/

# COPY，两种形式
# COPY [--chown=<user>:<group>] <src>... <dest>
# COPY [--chown=<user>:<group>] ["<src>",... "<dest>"]
# <user> ：linux用户
# <group>：linux组
# --chown不适用于 Windows 容器
# 用法同 ADD
COPY ${JAR_FILE} app.jar

# VOLUME，挂载点，将其标记为保存来自本机主机或其他容器的外部挂载卷
VOLUME /myvol

# WORKDIR，工作目录
WORKDIR /app_test_demo

# CMD，三种形式
# CMD ["executable","param1","param2"]（执行形式，这是首选形式）
# CMD ["param1","param2"]（作为ENTRYPOINT 的默认参数）
# CMD command param1 param2（贝壳形式）
# Dockerfile 中只能有一条CMD指令， 如果列出多个，则只有最后一个CMD生效
# run 时运行
CMD sleep 60; java -jar app.jar $JAVA_OPTS

# RUN CMD ENTRYPOINT 三者区别参考
# https://www.jianshu.com/p/f0a0f6a43907

# ADD COPY 两者区别参考
# https://www.cnblogs.com/flyphper/p/14035214.html
```