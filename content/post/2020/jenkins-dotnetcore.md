---
layout: post
title: 手把手教你用Jenkins自动发布dotnet core网站
category: dotnet core
date: 2018-05-08
tags:
- dotnet core
- Jenkins
- docker
---
# Jenkins部分

首先,我们要有个Jenkins咯,下载链接:[https://jenkins.io/download/](https://jenkins.io/download/)

我们安装官网教程安装好jenkins,安装教程略....

嗯?不是说好手把手么?你妹的.

好好好,我们还是来手把手教程好了.

## 首先安装JDK8

添加安装源之后直接apt-get install就好,下面是ubuntu的安装命令,其他系统自己玩一下就好.

```sh

sudo add-apt-repository ppa:webupd8team/java

sudo apt-get update

sudo apt-get install oracle-java8-installer

```

## 下载jenkins.war + 启动Jenkins

下载链接:[http://mirrors.jenkins.io/war-stable/](http://mirrors.jenkins.io/war-stable/)

在这里面找最新的下载,我当前最新的应该是[2.107.2](http://mirrors.jenkins.io/war-stable/2.107.2/jenkins.war)

下载好了jenkins.war之后,在当前目录创建一个jenkins-home文件夹,设置JENKINS_HOME环境变量为jenkins-home(不设置也可以,默认在~/.jenkins)

```sh

wget http://mirrors.jenkins.io/war-stable/2.107.2/jenkins.war;
mkdir ~/jenkins-home;
export JENKINS_HOME=~/jenkins-home;
tmux;
java -jar jenkins.war

```

一般建议开个后台进程来跑jenkins,免得终端退出之后jenkins就死掉了.

所以上面我先打开了tmux之后再跑java -jar jenkins.war.

如下图:
![jenkins启动](http://qiniu.house2048.cn/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-05-06%20%E4%B8%8B%E5%8D%886.53.24.png)

接着留意一下initialAdminPassword的输出

```log

Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

XXXXXXXXXXXXXX

This may also be found at: /root/jenkins-home/secrets/initialAdminPassword
```

这个时候访问当前主机的8080端口已经可以看到jenkins正在启动了,稍等片刻就可以看到jenkins登录页.

这个时候把上面的XXXXXXXXXXXXXX复制出来,输进去点击继续配置jenkins账号密码信息之类的.

![配置jenkins](http://qiniu.house2048.cn/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-05-06%20%E4%B8%8B%E5%8D%886.58.58.png)

接着安装默认插件.

![安装插件](http://qiniu.house2048.cn/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-05-06%20%E4%B8%8B%E5%8D%887.00.12.png)

这里估计也要等几分钟不等,看你的机器性能和网络速度.

安装好了之后会进入配置登录账号密码,安装提示配置就完事.

最后进入jenkins页面是这样的.
![jenkins](http://qiniu.house2048.cn/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-05-06%20%E4%B8%8B%E5%8D%887.02.42.png)

到现在我们已经把jenkins跑起来了,也有了一些常用的插件.

我们先去把dotnet core docker 编译发布相关的东西弄好之后再回来继续做jenkins任务.

## dotnet core docker 打包

在项目目录下新建Dockerfile文件,内容如下:

```docker

FROM microsoft/aspnetcore-build:2.0 AS build-env
WORKDIR /app

# copy csproj and restore as distinct layers
COPY *.csproj ./
RUN dotnet restore

# copy everything else and build
COPY . ./
RUN dotnet publish -c Release -o out

# build runtime image
FROM microsoft/aspnetcore:2.0
WORKDIR /app
COPY --from=build-env /app/out .
ENTRYPOINT ["dotnet", "你的dotnet core程序.dll"]

```

这个Dockerfile基本就是把当前目录的文件拷贝到aspnetcore-build镜像中,再里面编译好之后再发布到aspnetcore:2.0镜像中,

最后指定运行你的dotnet core程序

来源:[https://github.com/DaoCloud/dotnet-docker-samples](https://github.com/DaoCloud/dotnet-docker-samples)

## docker build + run 脚本(非必须,可以使用jenkins中脚本编译替代)

以[HouseCrawler.Web](https://github.com/liguobao/58HouseSearch/blob/master/HouseCrawler.Core/HouseCrawler.Web/)为例,

```sh

#!/bin/sh
image_version=`date +%Y%m%d%H%M`;
echo $image_version;
cd ~/code/58HouseSearch/HouseCrawler.Core/HouseCrawler.Web;
git pull --rebase origin master;
docker stop house-web;
docker rm house-web;
docker build -t house-web:$image_version .;
docker images;
docker run -p 8080:80 -v ~/docker-data/house-web/appsettings.json:/app/appsettings.json -v ~/docker-data/house-web/NLogFile/:/app/NLogFile  --restart=always --name house-web -d house-web:$image_version;
docker logs house-web;

```

通过上面这个build+run脚本,我们已经把dotnet core程序编译好了,并且打包成了docker images,还直接跑起来了.

但是我们想要的应该是自动化编译部署,而且上面我们都把jenkins跑起来了,所以....

## jenkins job配置

### 新建Job

打开jenkins首页,左侧选择"新建任务"(newJob),如下图:

![newJob](http://qiniu.house2048.cn/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-05-06%20%E4%B8%8B%E5%8D%889.10.00.png)

给新的job取个名字,然后选择"构建自由风格的软件项目",如图:

![构建自由风格的软件项目](http://qiniu.house2048.cn/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-05-06%20%E4%B8%8B%E5%8D%889.12.27.png)

### 添加源码仓库

确认之后进入Job配置页面,源码管理里面选择git,如图:
![源码管理](http://qiniu.house2048.cn/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-05-06%20%E4%B8%8B%E5%8D%889.14.06.png)

如果git仓库是需要权限的话需要配置一下权限,我一般简单粗暴直接把jenkins主机的公钥添加到git仓库里面,所以这里直接配置成'From the Jenkins master ~/.ssh',也可以用账号密码访问等等的.

![git仓库权限配置](http://qiniu.house2048.cn/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-05-06%20%E4%B8%8B%E5%8D%889.16.09.png)

"Branch Specifier (blank for 'any')	"默认master分支,根据自己的需求填入不同的分支.

构建触发器和构建环境先跳过,我们不管,待会弄.

### 构建

点击"添加构建步骤",选择"Execute shell",然后能看到如下图:
![Execute shell](http://qiniu.house2048.cn/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-05-06%20%E4%B8%8B%E5%8D%889.22.45.png)

还记得我们上一步的脚本么?修改一下源码路径再放进去.

```sh
# 切换到源码目录,对应在jenkins-home的workspace下面
cd ~jenkins-home/workspace/项目名称/Dockerfile所在目录;
image_version=`date +%Y%m%d%H%M`;
echo $image_version;
# 停止之前的docker container
docker stop house-web;
# 删除这个container
docker rm house-web;
# build镜像并且打上tag
docker build -t house-web:$image_version .;
docker images;
# 把刚刚build出来的镜像跑起来
docker run -p 8080:80 -v ~/docker-data/house-web/appsettings.json:/app/appsettings.json -v ~/docker-data/house-web/NLogFile/:/app/NLogFile  --restart=always --name house-web -d house-web:$image_version;
docker logs house-web;
```

如果jenkins主机和程序运行主机不在一台机器上,建议直接在把上面的脚本放在运行主机上,命名成 start_XXX.sh.

上面的命令直接就是成了

```sh
ssh username@发布主机的IP '~/start_XXX.sh'
```

ps:记得在jenkins主机配置[ssh免登陆](https://blog.csdn.net/wind520/article/details/38421359)

### 构建触发器

构建触发器就是我们选择什么时候来触发构建任务,有几种方案可以做.

1. 使用 Build periodically,定时 or 隔N久去拉一次代码构建
2. Poll SCM：定时检查源码变更（根据SCM软件的版本号）,如果有变化就去执行构建
3. GitHub hook trigger for GITScm polling 或者其他Git平台提供的webhook
4. 安装Generic Webhook Trigger插件之后,使用其他平台的webhook来触发构建任务.

我这里选择第4种方案,安装Generic Webhook Trigger插件,下面马上回告诉你为什么这样做的.

Generic Webhook Trigger插件在"系统管理-管理插件-可选插件"里面直接搜"Generic Webhook Trigger"安装就可以.

从上一步的构建步骤里面的脚本中我们就知道,其实我们现在要不就在jenkins主机上docker build,要不就在发布目标主机上build,

build过程比较慢而且还会产生镜像在本机or目标主机上,docker images也没有被管理起来.

有什么好的办法么?嗯,还真有.直接用阿里云"容器镜像服务"来构建镜像

### 使用阿里云-容器镜像服务

首先登录阿里云,然后进入容器镜像服务,地址是[https://cr.console.aliyun.com/](https://cr.console.aliyun.com/)

首次进入估计需要创建一个命名空间,一般用公司名或者你的名字就完事.

接着选择"创建镜像仓库".

![创建镜像仓库](http://qiniu.house2048.cn/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-05-06%20%E4%B8%8B%E5%8D%889.51.25.png)

选地区-选命名空间-填仓库名称(就是镜像名称)-填摘要-设置代码源(支持GitHub/阿里云code/Bitbucket/私有Gitlab/本地Git等等,给个授权就完事)

![选地区](http://qiniu.house2048.cn/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-05-06%20%E4%B8%8B%E5%8D%889.54.10.png)

构建设置选择"代码变更时自动构建镜像",然后选一下构建分支为你想要的分支,填入Dockerfile在源码中的路径,然后保存
![构建分支](http://qiniu.house2048.cn/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-05-06%20%E4%B8%8B%E5%8D%889.57.50.png)

接着我们进入管理平台看一下.

![aliyun-构建](http://qiniu.house2048.cn/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-05-06%20%E4%B8%8B%E5%8D%8810.00.51.png)

点击一下"立即构建",然后查看一下日志.
![build 日志](http://qiniu.house2048.cn/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-05-06%20%E4%B8%8B%E5%8D%8810.02.00.png)

![构建成功](http://qiniu.house2048.cn/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-05-06%20%E4%B8%8B%E5%8D%8810.02.49.png)

这个时候,我们用docker pull registry-internal.cn-hangzhou.aliyuncs.com/你的命名空间/你的镜像名称 就可以拉到这个阿里云build成功的镜像了.

镜像build的问题解决了,那么我们怎么自动把镜像发布到我们的运行主机呢?

这时候webhook又出来了.

### jenkins webhook触发配置

我们看阿里云镜像构建服务里面,有一项是webhook的,官方介绍在这里:[阿里云-webhook管理](https://help.aliyun.com/document_detail/60949.html?spm=5176.8351553.0.0.645319912fjxim)

![阿里云-webhook管理](http://qiniu.house2048.cn/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-05-06%20%E4%B8%8B%E5%8D%8810.08.45.png)

这里就需要填入我们的webhook地址,还记得前面我无端端选择的第四种方案,然后让大家跟着安装的Generic Webhook Trigger插件么?

我们就是用这货来为我们提供webhook API.

理一下流程:

git仓库代码变化 ->阿里云容器构建服务启动 -> 构建好镜像之后触发webhook -> jenkins收到阿里云的webhook之后触发job执行部署脚本 ->部署脚本使用阿里云镜像run起来 ->完事.

我们继续配置Generic Webhook Trigger.

Generic Webhook Trigger支持的命名触发URL格式是这样的:

``` http
http://jenkins登录用户名:token授权码@jenkins IP:8080/generic-webhook-trigger/invoke?token=触发器名称
```

jenkins登录名和token在"账号-设置-API Token-Show API Token..."里面能看到,找出来之后填到上面去就可以.

最后一个token参数其实就是"构建触发器"中"触发远程构建"的参数,建议使用job名字.这里的配置大概是这样的:

![触发远程构建](http://qiniu.house2048.cn/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-05-06%20%E4%B8%8B%E5%8D%8810.21.13.png)

最后我们还需要在jenkins全局安全设置中取消勾选“防止跨站点请求伪造（Prevent Cross Site Request Forgery exploits)"选项,这样阿里云webhook才能过得来.

手动在浏览器中访问一下http://jenkins登录用户名:token授权码@jenkins IP:8080/generic-webhook-trigger/invoke?token=触发器名称
如果对应的jenkins Job能正常开始执行,说明整个流程已经ok了.

最后我们回到上面"阿里云-容器镜像服务-对应镜像仓库-webhook-添加记录"
![webhook-添加记录](http://qiniu.house2048.cn/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-05-06%20%E4%B8%8B%E5%8D%8810.27.02.png)

PS:webhook名称不要带特殊字符or "-"之类的,不然一直保存失败而且还不会提示你是因为名字不合法,下午被这个坑了半个小时.

到这里,我们基本大功告成了.

最后我们再改一下jenkins的脚本,不在本地build docker了,直接拿阿里云镜像服务构建出来的镜像跑就可以.

```sh
# 停止之前的docker container
docker stop house-web;
# 删除这个container
docker rm house-web;
docker pull 你的阿里云镜像地址;
# 把刚刚build出来的镜像跑起来
docker run --restart=always --name 你的contianer名称 你的阿里云镜像地址;

```

### 总结一下我们做了什么

1. 搭建jenkins
2. 编写Dockerfile文件,直接编译发布+打包成docker镜像+部署脚本
3. 使用阿里云-容器构建服务构建docker镜像,构建成功后使用webhook通知jenkins
4. 配置jenkins webhook触发器,触发部署脚本
5. 其他项目/语言其实也基本一样的操作,区别只在于Dockerfile的编写
6. 完事...