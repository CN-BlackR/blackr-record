# docker-compose

>
> <b>官方连接</b>
>
> [本文列举了部分常用的配置，完整版跳转到官方文档](https://docs.docker.com/compose/compose-file/)
>
> [推荐版本：docker-compose和docker](https://docs.docker.com/compose/compose-file/compose-versioning/#compatibility-matrix)

|  Compose file format   | Docker Engine release  |
|  ----  | ----  |
| Compose specification  | 19.03.0+ |
| 3.8  | 19.03.0+ |
| 3.7  | 18.06.0+ |
| 3.6  | 18.02.0+ |
| 3.5  | 17.12.0+ |
| 3.4  | 17.09.0+ |
| 3.3  | 17.06.0+ |
| 3.2  | 17.04.0+ |
| 3.1  | 1.13.1+  |
| 3.0  | 1.13.0+  |
| 2.4  | 17.12.0+ |
| 2.3  | 17.06.0+ |
| 2.2  | 1.13.0+  |
| 2.1  | 1.12.0+  |
| 2.0  | 1.10.0+  |

```yml
# https://docs.docker.com/compose/compose-file/compose-file-v3/#context
version: "3.7"

services:
  # 镜像信息
  appDemo1:
    # 镜像名称；如果镜像不存在，会尝试拉取它，除非您还指定了build，在这种情况下，它会使用指定的选项构建它并使用指定的标签对其进行标记。
    image: demo
    # 执行命令
    command: sh -c "pwd"
    # 端口映射
    ports:
      # 3001宿主机端口；3000容器端口
      - 3001:3000
    # 构建阶段
    build:
      # Dockerfile文件夹路径
      context: .
      # Dockerfile
      dockerfile: Dockerfile
      # 构建时环境变量
      args: 
        testArg1: 1
        testArg2: 2
      # 缓存加速
      cache_from:
        - alpine:latest
      # 设置网络容器,none禁用
      network: myNetwork
      # 设置/dev/shm虚拟共享内存，默认为内存的一半，df -h查看，可配置值单位：2b、1024kb、2048k、300m、1gb
      # shm是啥？片面的介绍：https://blog.csdn.net/jacson_bai/article/details/41278565
      shm_size: 2gb
      # 多阶段构建:https://www.linuxea.com/2314.html
      # target: 
      # 部署配置
      deploy:
        # 重启策略
        restart_policy: 
          # none、on-failure、any；默认值: any)
          condition: on-failure
          # 重启间隔时间
          delay: 5s
          # 重启重试次数
          max_attempts: 3
          # 重启后，判断存活时间，超过则重启成功，未超过则继续尝试重启（默认立即决定）
          window: 120s
    # 从文件中获取运行时环境变量；期望 env 文件中的每一行都具有VAR=VAL格式。以开头的行#被视为注释并被忽略。空行也被忽略。列表中的文件是自上而下处理，重复值会被最后一个覆盖
    env_file: 
      - ./common.env
      - /opt/runtime_opts.env
    # 运行时环境变量
    environment:
      - RACK_ENV=development
      - SHOW=true
      - SESSION_SECRET
    # 开放端口。公开端口而不将它们发布到主机 - 它们只能被链接服务访问。只能指定内部端口。
    expose:
      - "3000"
      - "8000"
    # 添加主机名映射
    extra_hosts:
      - "somehost:162.242.195.82"
      - "otherhost:50.31.209.229"
    # 健康检查，https://docs.docker.com/engine/reference/builder/#healthcheck
    healthcheck:
      # 禁用健康检查
      disable: true
      # 健康检查命令，根据实际情况修改
      test: curl -f https://localhost || exit 1
      # 检查间隔时间
      interval: 1m30s
      # 单次运行检查花费的时间
      timeout: 10s
      # 重试次数
      retries: 3
      # 应用程序启动的初始化时间。启动期间失败的运行状况检查不计算在内。默认值为0秒
      start_period: 40s
    # 链接到另一个服务中的容器。要么指定服务名称和链接别名 ( "SERVICE:ALIAS")，要么只指定服务名称。
    # 它最终可能会被删除。除非您绝对需要继续使用它，否则我们建议您使用 networks(https://docs.docker.com/compose/networking/) 来促进两个容器之间的通信
    # networks基础配置介绍（https://docs.docker.com/compose/compose-file/compose-file-v3/#cgroup_parent）
    links:
      - "db"
      - "db:database"
      - "redis"
    # 日志记录
    logging:
      # 日志记录驱动："json-file"、"syslog"、"none"；默认"json-file"
      # 只有json-file和journald驱动程序才能直接从docker-compose up和获取日志docker-compose logs。使用任何其他驱动程序不会打印任何日志。
      driver: syslog
      options:
        syslog-address: "tcp://192.168.0.42:123"
        # 最大存储大小，超过产生一个日志文件
        max-size: "200k"
        # 最大文件数，旧日志文件将被删除
        max-file: "10"
    # 磁盘（卷）映射
    # [SOURCE:]TARGET[:MODE]  SOURCE可以是主机路径或卷名。TARGET是安装卷的容器路径。标准模式:ro只读、rw读写（默认）。
    # 详细配置:https://docs.docker.com/compose/compose-file/compose-file-v3/#long-syntax-3
    volumes:
      - /opt/data:/var/lib/mysql:rw
    # 自动重启
    # "no"、always、on-failure、unless-stopped 默认："no"
    restart: always
```