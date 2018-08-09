##Docker 安装 YouTrack
[官网安装地址](https://www.jetbrains.com/help/youtrack/standalone/youtrack-docker-installation.html#pull-image)

###1. 拉去YouTrack镜像
从[镜像服务器](https://hub.docker.com/r/jetbrains/youtrack/)查看最新的镜像版本号

```
docker pull jetbrains/youtrack:<version>
```

###2. 创建并配置本地路径
在安装之前, 我们需要在本地创建文件夹来保存YouTrack的数据库, 配置文件, 日志, 备份文件, 文件夹名称分别为data, conf, logs, backups, 并将文件夹路径作为磁盘卷轴传递给YouTrack容器, 不然, 然我们移除YouTrack容器时, 数据就会全部丢失.</br>

- data — 保存YouTrack的数据库, 第一次安装必须为空目录</br>
- conf — 保存YouTrack的配置文件</br>
- logs — 保存YouTrack的日志文件</br>
- backups - 保存YouTrack的本分文件</br>

以上文件夹必须对于运行YouTrack的账号, 必须是可访问的.YouTrack使用non-root 账号 `13001:13001` (`group:id`, respectively).
所以, 文件夹创建时需要加上权限设置:

```
mkdir -p -m 750 <path to data directory> <path to logs directory> <path to conf directory>
<path to backups directory>

chown -R 13001:13001 <path to data directory> <path to logs directory> <path to conf
directory> <path to backups directory>
```

###3. 运行YouTrack容器

使用以下命令运行Youtrack容器, 并配置Youtrack的数据卷轴和端口:

```
docker run -it --name <youtrack-server-instance> \
-v <path to data directory>:/opt/youtrack/data \
-v <path to conf directory>:/opt/youtrack/conf \
-v <path to logs directory>:/opt/youtrack/logs \
-v <path to backups directory>:/opt/youtrack/backups \
-p <port on host>:8080 \
jetbrains/youtrack:<version>
```

* -it 将docker的输入输出添加到当前命令窗口
* --name 容器名称
* -v 挂载卷轴
* -p 映射端口

###4. 配置YouTrack
执行以上命令后, YouTrack正常运行, 打开网址后, 按照安装向导提示, 一步一步执行完毕; 之后以administrator身份登录, 打开 administration menu > Global Settings进行相关配置

#### 命令行配置YouTrack
出于自动安装的需求, 你也可以跳过向导, 但是, 此时YouTrack是按照默认参数运行. 英雌, 你需要单独配置面向用户的Youtrack URL地址.
要跳过配置向导, 需要在运行YouTrack之前运行配置命令:
```
docker run --rm -it
-v <path to conf directory>:/opt/youtrack/conf \
-v <path to logs directory>:/opt/youtrack/logs \
jetbrains/youtrack:<version> \
configure \
-J-Ddisable.configuration.wizard.on.clean.install=true \
--base-url=http://youtrack.mydomain.com:XXXX
```

#### 配置YouTrack URL

如果需要修改正在运行的Youtrack根URL, 建议通过Admin menu > Global Settings进行修改, 当然, 也可以用一下命令实现:

```
停止YouTrack:
 docker exec <containerId> stop

运行以下命令:
docker run --rm -it
-v <path to conf directory>:/opt/youtrack/conf \
-v <path to logs directory>:/opt/youtrack/logs \
jetbrains/youtrack:<version> \
configure --base-url=https://<YouTrack service baseURL>:XXXX

运行YouTrack:
 docker start <containerId>


```
查看全部容器, 运行` docker ps -a`


###Note
如果配置向导提示`opt/youtrack/conf`不为空,
可执行以下命令,找到conf目录清空内容:

```
sudo docker exec -it containerID /bin/bash 
```

或者

```
docker run -it --rm jetbrains/youtrack:<version> bash
```