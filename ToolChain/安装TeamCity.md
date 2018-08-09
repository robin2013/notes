#Docker 安装配置 TeamCity

确定本机已安装docker运行环境, docker-compose环境

## 安装
[参见](https://blog.agchapman.com/setting-up-a-teamcity-build-environment-using-docker/)

###创建文件夹
创建一个teamcity的根目录, 并创建 `agent` `datadir`  `log` `config` 四个文件夹:

* agent 外围Agent安装目录
* datadir teamcity数据文件夹
* log 日志文件夹
* config docker内部Agent配置

### 创建安装文件
在根目录创建一个docker-compose.yml文件, 内容如下:

```
version: '2'
services:
  server:
    image: 'jetbrains/teamcity-server'
    volumes:
      - ~/ToolChain/teamcity/datadir:/data/teamcity_server/datadir
      - ~/ToolChain/teamcity/log:/data/teamcity/logs
    ports:
      - 8891:8111
    environment:
      - TEAMCITY_SERVER_MEM_OPTS="-Xmx750m"
      
  agent:
    image: 'jetbrains/teamcity-agent'
    environment:
      - SERVER_URL=server:8111
    volumes:
      - ~/ToolChain/teamcity/config:/data/teamcity_agent/conf
```

###运行安装
`docker-compose up -d`

不出意外的话, 安装完成后, 会提示访问`localhost:8111`打开teamcity, 其实这个网址不一定正确, 上边配置中, 笔者修改了映射端口,所以访问 `localhost:8891`

##配置
###配置管理员
打开`localhost:8891`, 会要求配置数据库,你也可以选择MySql, 可能会提示缺少Driver.

1. 运行`docker exec -it <ContainerID> /bin/bash`进入Container
2. 切换到页面提示的路径下[我没记住路径], 执行:

```
curl -O https://cdn.mysql.com//Downloads/Connector-J/mysql-connector-java-8.0.11.tar.gz
```
或者

```
curl -O https://cdn.mysql.com//Downloads/Connector-J/mysql-connector-java-5.1.46.tar.gz
```
下载一个合适的驱动(我没链接成功, 总是提示链接失败);当然, 最简单的就是使用内置的数据库.
 
###创建工程
 
选择完数据库后,创建管理员账户, 然后就可以创建工程.

#### Project Create
笔者的工程保存在Gitlab上, 公司的网络需要认证, 所以选择`From a repository URL`一直无法通过认证, (Github可以),
所以就先用`Manually`模式代替.

### 上传SSH
[参见](https://d-fens.ch/2015/09/02/nobrainer-use-ssh-key-on-jetbrains-teamcity/)
通过ssh的方式可以解决https不能认证的问题

####创建秘钥
执行以下命令, 创建秘钥
```
ssh-keygen -t rsa -C "your.email@example.com" -b 4096
```

复制秘钥的内容:
```
MacOS
pbcopy < ~/.ssh/id_rsa.pub
```

```
Linux
xclip -sel clip < ~/.ssh/id_rsa.pub
```

```
Windows
type %userprofile%\.ssh\id_rsa.pub | clip
```
并将公钥内容填写到GitLab的个人`Setting` -> `SSH Keys`中

#### 上传私钥

1. 点击`Projects`
2. 选择`SSH Keys`
3. 点击`Upload SSH Key`
4. 选择上一步创建的`id_rsa`文件, 并上传

### 编辑项目
1. 点击你刚建立的项目，然后点右下角的箭头, 选择 `Edit Settings`,
2. 创建` VCS Roots`, `Type of VCS`选择`Git`
3. `Fetch URL` 栏填入 SSH:// 的地址, UserName 和 Password 为空, 这样才能启用ssh key
4. `Authentication method:`选择`Uploaded Key`
5. `Uploaded Key: `选择刚刚上传的`id_rsa`的文件
6. 补充其他必填项并保存

然后添加Build Steps即可, 具体步骤[参见](https://wysockikamil.com/teamcity-for-ios-project/)

###重新创建Agent
[参见](https://confluence.jetbrains.com/display/TCD18//Setting+up+and+Running+Additional+Build+Agents#SettingupandRunningAdditionalBuildAgents-InstallingviaZIPFile)
由于docker是linux, 内置Agent是不能编译iOS文件的, 这就需要我们本机Agent
1. 点击`Agents`
2. 选择`Install Build Agents`
3. 选择`Zip file distribution`下载文件包
4. 解压文件, 并将文件夹`buildAgent`拷贝到`~/ToolChain/teamcity/agent`(笔者的目录)下
5. 将`buildAgent/conf/`下的`buildAgent.dist.properties`命名为`buildAgent.properties`
6. 用xCode编辑`buildAgent.properties`, 修改`serverUrl=http://localhost:8891/`(笔者的映射端口)
7. 切换到`bin`目录, 运行`./agent.sh start`
8. 打开TeamCity, 选择`Agents`, 在`Unauthorized`下就可以看到刚才创建的Agent
