---
layout: post
title: 可能是全网首个支持阿里云Elasticsearch Xapck鉴权的Skywalking
category: API
date: 2018-06-18
tags:
- APM
- Skywalking
- Elasticsearch
---

# 可能是全网首个支持阿里云Elasticsearch Xapck鉴权的Skywalking

对Skywalking有兴趣的同学参见:[年轻人的第一个APM-Skywalking](https://zhuanlan.zhihu.com/p/45084693)

之前在搭建Skywalking的时候发现,官方Skywalking 5.X并支持有鉴权的Elasticsearch.

而我司有其他需求已经购买了阿里云的Elasticsearch,咨询过阿里云技术支持后他们表示并不能去掉鉴权,所以只好自己想办法了.

又在Skywalking技术群问了一圈,有其他人也遇到过类似的问题,但是最后还是选择自建ES了.

实在不想自己再浪费精力去搭建ES了,还是觉得可以尝试一下别的方案.

然后咨询了一下wusheng大大之后,他说可以自己尝试换一个支持XPack鉴权的Client,应该没什么太大的问题.

于是就开始了"填坑"之旅.

## 首先是引入x-pack-transport支持

apm-collector/apm-collector-component/client-component/pom.xml 

```xml

        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>x-pack-transport</artifactId>
            <version>${elasticsearch.client.version}</version>
        </dependency>

        <repositories>
        <repository>
            <id>elasticsearch-releases</id>
            <url>https://artifacts.elastic.co/maven</url>
            <releases>
                <enabled>true</enabled>
            </releases>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
    </repositories>
```

接着在 ...in/java/org/apache/skywalking/apm/collector/client/elasticsearch/ElasticSearchClient.java

加入PreBuiltXPackTransportClient的初始化

```java

 private final String securityUser;


 private PreBuiltXPackTransportClient initXPackClient() {
        Settings settings = Settings.builder()
                .put("cluster.name", clusterName)
                .put("xpack.security.transport.ssl.enabled", false)
                .put("xpack.security.user", securityUser)
                .put("client.transport.sniff", false).build();
        return new PreBuiltXPackTransportClient(settings);
     }
     private PreBuiltTransportClient initClient() {
        Settings settings = Settings.builder()	        Settings settings = Settings.builder()
            .put("cluster.name", clusterName)	                .put("cluster.name", clusterName)
            .put("client.transport.sniff", clusterTransportSniffer)	                .put("client.transport.sniff", clusterTransportSniffer)
            .build();	                .build();
        return new PreBuiltTransportClient(settings);
    }

     // 新增 private final String securityUser;
    // 判断这个变量是不是null或者空字符串,如果是就默认初始化,不是则使用initXPackClient初始化
    // 改一下initialize 方法

    private final String securityUser;


     @Override
    public void initialize() throws ClientException {
        if (securityUser == null || "".equals(securityUser)) {
            client = initClient();
        } else {
            client = initXPackClient();
        }

```

然后还要把apm-collector/pom.xml的elasticsearch.client.version 版本改成5.3.3.

改完之后因为5.3.3和原来5.5.0有点不一样,需要修改一下很几个地方的代码.

这时候建议直接使用IDEA build 一下,哪里报错改哪里就好.

主要都是 searchResponse.getHits().totalHits 改成searchResponse.getHits().totalHits()

神奇发现5.5.0版本的ES Client把5.3.3的searchResponse.getHits().totalHits() 方法改成了属性.

不经感慨都是人才啊...

别的一些基本都是引入 import org.elasticsearch.action.bulk.byscroll.BulkByScrollResponse;

全部代码在这里:[liguobao/incubator-skywalking](https://github.com/liguobao/incubator-skywalking/commit/ae3d43bc7a65a1b955e6bdf75a2356c8ecf53f28)

完整改好的代码在[liguobao/incubator-skywalking](https://github.com/liguobao/incubator-skywalking/commit/ae3d43bc7a65a1b955e6bdf75a2356c8ecf53f28)

同时配置的时候添加一下 securityUser参数,如果ES有鉴权就传入,没有的话就不传,这样就达到鉴权和不鉴权两种需求的兼容了.

## 支持xpack的docker部署方案

完整原文链接:[Skywalking-Dcoker for ES xpack 镜像](https://github.com/liguobao/skywalking-docker/tree/master/5.x/standalone/all-in-one-xpack)



[![Docker Build Status](https://img.shields.io/docker/build/liguobao/skywalking-docker.svg)](https://hub.docker.com/r/liguobao/skywalking-docker/)
[![Docker Pulls](https://img.shields.io/docker/pulls/liguobao/skywalking-docker.svg)](https://hub.docker.com/r/liguobao/skywalking-docker/)

## Dockerfile说明

apache-skywalking-apm-incubating.tar.gz为支持ES X-Pack修改后打包出来的压缩包,此仓库没有这个文件的.

可以去QQ群:Apache SkyWalking交流群(392443393)群文件中下载apache-skywalking-apm-incubating-xpack.tar.gz

或者自行编译[liguobao/incubator-skywalking/tree/5.x](https://github.com/liguobao/incubator-skywalking/tree/5.x) 此版本的源码.

编译步骤:

```sh
# Prepare git, JDK8 and maven3
git clone https://github.com/liguobao/incubator-skywalking.git
cd incubator-skywalking/
git checkout 5.x
#Switch to the tag by using git checkout [tagname] (Optional, switch if want to build a release from source codes)
git submodule init
git submodule update
Run ./mvnw clean package -DskipTests
#All packages are in /dist.(.tar.gz for Linux and .zip for Windows).
```

Docker 镜像名称:[liguobao/skywalking-docker](https://hub.docker.com/r/liguobao/skywalking-docker/)

## 拉取镜像（Pull Image）:
```docker pull liguobao/skywalking-docker:5.0.RC2.xpack```

## 运行镜像（Run）for ES xpack:
- ```docker run -p 8080:8080 -p 10800:10800 -p 11800:11800 -p 12800:12800 -e ES_CLUSTER_NAME=elasticsearch -e ES_ADDRESSES=192.168.2.96:9300 -e SECURITY_USER='elastic:password' -d liguobao/skywalking-docker:5.0.RC2.xpack```
- 使用浏览器访问```http://localhost:8080```即可.
- 日志挂载 ```-v /your/log/path:/apache-skywalking-apm-incubating/logs```

## 环境变量（Environment Variables）
- ```ES_CLUSTER_NAME```,```ES_ADDRESSES```:elasticsearch 地址和集群名称。注意：此处Elasticsearch地址中的端口务必是Elasticsearch TCP端口。
- ```SECURITY_USER```,```SECURITY_USER```:elasticsearch 的账号密码,使用X-Pack实现的,常见阿里云ES,格式为:'user:password'.此参数不传入或者传入'' ,默认使用没有授权的client.
- ```NAMING_BIND_HOST```,```NAMING_BIND_PORT```:OS real network IP(binding required),for agent to find collector cluster.
- ```BIND_HOST```,```REMOTE_BIND_PORT```:OS real network IP(binding required),for collector nodes communicate with each other in cluster. collectorN --(gRPC) --> collectorM
- ```AGENT_GRPC_BIND_PORT```:OS real network IP(binding required),for agent to uplink data(trace/metrics) to collector. agent--(gRPC)--> collector
- ```AGENT_JETTY_BIND_HOST```,```AGENT_JETTY_BIND_PORT```:OS real network IP(binding required), for agent to uplink data(trace/metrics) to collector through HTTP. agent--(HTTP)--> collector
-```UI_JETTY_BIND_HOST```,```UI_JETTY_BIND_PORT```:Stay in `0.0.0.0` if UI starts up in default mode.Change it to OS real network IP(binding required), if deploy collector in different machine.



## 与elasticsearch-shanghai-zone镜像配合使用请参考
- [wutang/elasticsearch-shanghai-zone](../../../elasticsearch-5.6.10-Zone-Asia-SH/README.md)
- [quick start](../5.x/quick-start/README.md)


## 后记

本来还打算把代码提给主仓库的,但是wusheng 大大说xpack客户端和Apache要求的授权有冲突,遂...

那就留着自己玩了.

拜...

