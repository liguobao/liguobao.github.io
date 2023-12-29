---
layout: post
title: 每周开源项目分享-年轻人的第一个OAuth2.0 Server:hydra
category: 每周开源项目分享
date: 2018-07-28
tags:
- Github
- Open Source
- hydra
- 每周开源项目分享
---

# 年轻人的第一个OAuth2.0 Server:hydra

[hydra](https://github.com/ory/hydra) 是什么呢?

- OpenID Connect certified OAuth2 Server - cloud native, security-first, open source API security for your infrastructure. Written in Go. SDKs for any language.

讲人话的话就是一个OAuth2.0的服务端框架咯,开箱即用.

OAuth是撒? QQ互联知道么?微信授权登录知道么?

扩展阅读:[理解OAuth 2.0:阮一峰](http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)

开源地址:[https://github.com/ory/hydra](https://github.com/ory/hydra)

文档地址:[https://www.ory.sh/docs/guides/master/hydra/](https://www.ory.sh/docs/guides/master/hydra/)

本文结束...

</br>
</br>
</br>
</br>
</br>
</br>
</br>
</br>
</br>
</br>
</br>
</br>
</br>
</br>
</br>
</br>
</br>
</br>
</br>
</br>
</br>
</br>
</br>
</br>
</br>
</br>
</br>
</br>
</br>
</br>
</br>
</br>
</br>
</br>
</br>
</br>

## 开始手把手教你跑hydra server

- 全程Docker部署,请自行准备相关环境.

## 准备PostgreSQL 数据库/MySQL数据库

hydra支持PostgreSQL/MySQL,任君选择.

官网指导教程使用的是PostgreSQL,下面我也抄过来了,同时提供MySQL的相关操作.

启动数据库啦!!!

哦,启动之前先创建一个docker 网络.

```sh
docker network create hydraguide
```

启动 PostgreSQL,如下

```sh
docker run \
  --network hydraguide \
  --name ory-hydra-example--postgres \
  -e POSTGRES_USER=hydra \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=hydra \
  -d postgres:9.6

```

或者启动MySQL

```sh
docker run -p 3306:3306 \
 --network hydraguide \
 --name hydra-mysql --restart=always \
 -v ~/docker-data/hydra-mysql/data/:/var/lib/mysql \
 -e MYSQL_ROOT_PASSWORD=123 -d mysql:5.7
```

启动好了自行验证一下数据库是不是正确启动和能连接上去了.

## 准备hydra相关环境变量

```sh
# The system secret can only be set against a fresh database. Key rotation is currently not supported. This
# secret is used to encrypt the database and needs to be set to the same value every time the process (re-)starts.
# You can use /dev/urandom to generate a secret. But make sure that the secret must be the same anytime you define it.
# You could, for example, store the value somewhere.
$ export SYSTEM_SECRET=$(export LC_CTYPE=C; cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 32 | head -n 1)
#
# Alternatively you can obviously just set a secret:
# $ export SYSTEM_SECRET=this_needs_to_be_the_same_always_and_also_very_$3cuR3-._

# The database url points us at the postgres instance. This could also be an ephermal in-memory database (`export DATABASE_URL=memory`)
# or a MySQL URI.
$ export DATABASE_URL=postgres://hydra:secret@ory-hydra-example--postgres:5432/hydra?sslmode=disable

# MySQL的配置,host.docker.internal为宿主机IP,mysql容器的内部IP或者hydra-mysql也可以用

$ export DATABASE_URL=mysql://root:123@tcp(host.docker.internal/mysql容器的内部IP/hydra-mysql:3306)/hydra?parseTime=true

```

- SYSTEM_SECRET 是hydra启动时加密数据库使用的,Mac/Linux直接使用上面的方法设置即可,windows环境下设置一下环境变量?大概是这样.

- DATABASE_URL 是数据库连接配置,postgres和mysql 二选一即可.

## 执行迁移数据库脚本

hydra自带的,直接执行即可.

```sh
docker run -it --rm \
  --network hydraguide \
  oryd/hydra:v1.0.0-beta.5 \
  migrate sql $DATABASE_URL
```

正常执行的话,应该如下:

```log
Applying `client` SQL migrations...
Applied 0 `client` SQL migrations.
Applying `oauth2` SQL migrations...
Applied 0 `oauth2` SQL migrations.
Applying `jwk` SQL migrations...
Applied 0 `jwk` SQL migrations.
Applying `consent` SQL migrations...
Applied 0 `consent` SQL migrations.
Migration successful! Applied a total of 0 SQL migrations.
Migration successful!
```

数据库好了,我们现在可以开始搞服务端了.

## 启动hydra 服务端

```sh
docker run -d \
  --name ory-hydra-example--hydra \
  --network hydraguide \
  -p 9000:4444 \
  -e SYSTEM_SECRET=$SYSTEM_SECRET \
  -e DATABASE_URL=$DATABASE_URL \
  -e OAUTH2_ISSUER_URL=https://localhost:9000/ \
  -e OAUTH2_CONSENT_URL=http://localhost:9020/consent \
  -e OAUTH2_LOGIN_URL=http://localhost:9020/login \
  oryd/hydra:v1.0.0-beta.5 serve
```

这里我们留意几个传入给容器的环境变量.

- OAUTH2_ISSUER_URL hydra所在的地址

- OAUTH2_CONSENT_URL 授权页面地址

- OAUTH2_LOGIN_URL 登录页面地址

假装大家都了解OAuth2.0的流程的情况下,其实这里就流程基本就是:

XX应用请求授权

-> 跳转到OAUTH2_LOGIN_URL地址

-> 登录成功

->跳转到OAUTH2_CONSENT_URL授权页面

-> 授权成功

->回调XX应用地址并且返回相关授权code/token

-> XX应用使用code/token获取用户信息或者其他操作

启动之后看一下logs是不是hydra是不是正常启动.

常见问题:"Could not fetch private signing key for OpenID Connect - did you forget to run \"hydra migrate sql\" or forget to set the SYSTEM_SECRET?" error="unexpected end of JSON input"

确认一下SYSTEM_SECRET有没有正常设置呀,实在不行直接在docker run的时候带入.

正常启动的话,日志如下:

```log

Thank you for using ORY Hydra!

Take security seriously and subscribe to the ORY newsletter. Stay on top of new patches and security insights.

>> Subscribe now: http://eepurl.com/di390P <<
time="2018-08-09T10:23:50Z" level=info msg="Connected to SQL!"
time="2018-08-09T10:23:50Z" level=info msg="JSON Web Key Set hydra.openid.id-token does not exist yet, generating new key pair..."
time="2018-08-09T10:23:51Z" level=info msg="Setting up Prometheus middleware"
time="2018-08-09T10:23:51Z" level=info msg="Transmission of telemetry data is enabled, to learn more go to: https://www.ory.sh/docs/guides/latest/telemetry/"
time="2018-08-09T10:23:51Z" level=info msg="Detected local environment, skipping telemetry commit"
time="2018-08-09T10:23:51Z" level=info msg="Detected local environment, skipping telemetry commit"
time="2018-08-09T10:23:51Z" level=info msg="JSON Web Key Set hydra.https-tls does not exist yet, generating new key pair..."
time="2018-08-09T10:23:55Z" level=info msg="Setting up http server on :4444"
```

这时候去访问:[https://localhost:9000/.well-known/jwks.json](https://localhost:9000/.well-known/jwks.json) 

理论上是能正常输出结果的.

这个时候说明我们的hydra已经正常跑起来了.

## 登录/授权样例网站启动

```sh
docker run -d \
  --name ory-hydra-example--consent \
  -p 9020:3000 \
  --network hydraguide \
  -e HYDRA_URL=https://ory-hydra-example--hydra:4444 \
  -e NODE_TLS_REJECT_UNAUTHORIZED=0 \
  oryd/hydra-login-consent-node:v1.0.0-beta.5
```

在上面我们提过,XX应用请求授权的时候,首先是跳转到统一登录页面,

这个本质是是一个统一用户中心的应用,需要我们自行开发的.

hydra官方提供一个样例给我们来测试用,node.js写的.

项目地址:[https://github.com/ory/hydra-login-consent-node](https://github.com/ory/hydra-login-consent-node)

这里就是在启动这个登录/授权样例网站了.

PS:我用dotnet core也实现了一套完整的用户中心授权网站,过几天空了整理一下开源发出来.

## 创建oauth client 客户端

```sh
docker run --rm -it \
  -e HYDRA_URL=https://ory-hydra-example--hydra:4444 \
  --network hydraguide \
  oryd/hydra:v1.0.0-beta.5 \
  clients create --skip-tls-verify \
    --id facebook-photo-backup \
    --secret some-secret \
    --grant-types authorization_code,refresh_token,client_credentials,implicit \
    --response-types token,code,id_token \
    --scope openid,offline,photos.read \
    --callbacks http://127.0.0.1:9010/callback
```

没什么说的,留意一下callbacks地址即可.

其实就是XX互联里面的XX应用的一些信息.

## 测试hydra oauth整体授权流程

启动一个请求授权的APP,如下:

```sh
docker run --rm -it \
  --network hydraguide \
  -p 9010:9010 \
  oryd/hydra:v1.0.0-beta.5 \
  token user --skip-tls-verify \
    --port 9010 \
    --auth-url https://localhost:9000/oauth2/auth \
    --token-url https://ory-hydra-example--hydra:4444/oauth2/token \
    --client-id facebook-photo-backup \
    --client-secret some-secret \
    --scope openid,offline,photos.read
```

启动之后访问[http://127.0.0.1:9010/](http://127.0.0.1:9010/)

大概会看到:

```log
Welcome to the example OAuth 2.0 Consumer
This example requests an OAuth 2.0 Access, Refresh, and OpenID Connect ID Token from the OAuth 2.0 Server (ORY Hydra). To initiate the flow, click the "Authorize Application" button.

Authorize application
```

点击 Authorize application 立即调整到登录页面.

登录页其实就是我们上面启动的node.js的的登录页面,即:[http://localhost:9020/login?login_challenge=XXX](http://localhost:9020/login?login_challenge=XXX)

输入账号密码之后会跳到授权页面,即:[http://localhost:9020/consent?consent_challenge=XXXX](http://localhost:9020/consent?consent_challenge=XXXX)

授权选好了之后点击"" 允许授权,立即跳转回到回调地址,同时显示access token相关信息.

```txt
Access Token: SwMfFOSHEFpiChmBvRtFLTeaPzCh-TNEXtxTfibgmdw.AgqJrWyn1VlH4FouUucBJSDsmcwOGDI3cHpuy2sDrpI
Refresh Token: 48pXaTrBoXl9JxkweFgQV6frEYPwrkE6BaY8U5mymbo.ZuZe68sqX6wtRTk9k1cKBNPJxQzEBEb0G86tT_WVzCg
Expires in: 2018-08-09 11:46:14.9905228 +0000 UTC m=+3843.912315001
ID Token: eyJhbGciOiJSUzI1NiIsImtpZCI6InB1YmxpYzo3MzBhZDc4ZC04ODJmLTQzNzItYTRhMi05NTE2NDdlNTk0ZTciLCJ0eXAiOiJKV1QifQ.eyJhdWQiOlsiZmFjZWJvb2stcGhvdG8tYmFja3VwIl0sImF1dGhfdGltZSI6MTUzMzgxMTUyMSwiZXhwIjoxNTMzODE1MTc1LCJpYXQiOjE1MzM4MTE1NzUsImlzcyI6Imh0dHBzOi8vbG9jYWxob3N0OjkwMDAvIiwianRpIjoiOGFkMWE2ZTYtZWY3NC00MTM2LThmODUtMGU0N2FhYWYxZjY1Iiwibm9uY2UiOiJkcXBrcXlkaG1iZXZhdXNtYXJzdWxjcW0iLCJyYXQiOjE1MzM4MTE0MTMsInN1YiI6ImZvb0BiYXIuY29tIn0.euqVspiSeYvMonrwHSPxhfXaCOoYtfP5S5_dJLg6zeQ-Kw6rRJfQRh2ddMiaZBOHdRrQLGHouNSd5SCWP4DgKjr6eA4YKmiTNvDKt0ktIBfTROs5HKOIp9NHLSCL636m10lEVAGJEnL2jwVn5JeNjYmn4nRqOqPBfAxmqFYu-RuHk3HP4w9cKAK2tUBvwUkjH7PBkZ4MZI3AgvK985iPxZWkiyJAn4QPSAidenlQqQJXc7kpYpvP6wauk-nWxid6p0GRL1MozEV1Kok6Nqiw5BtEhuuC3Saijezr-G7Va6SwgTe731huzM6xRH_ovh2x4gayQu-qFX6bT8gjvLh6otQbqEa11nNc0gXIauKds2FF8mD65k9-tnFvbs3T7fJS6wu3LOm9VAtCB78CiTH92E7sbGXaQRC9nsB6LCCteoBPYa8e-dYZxXZHPdWP9tDNc3t2Zr1Lg5bljpWXmFcLllO6gSTqhKiT0otQaQgLDm9GvSeobEaCYRmgk50FdGz4z4Sngek6JJBWHNDo16zuJMScLxIdUfhK9LtLpIsL7w7F01GRMkcriowloRM85qO3V-Dq6REY6VzAe3OkT3_0bxsbU_fzFEIbpDcXdq8hchkEq3aAp48dqXb0WE4R7Iwl4JhjDKiQFxP4-Wk5rPqRyRs7rWiDUxS9v29c88pXd6E
```

这个时候我们使用Access Token去调用userinfo API,即可正常获取到用户信息.

```sh
curl -X GET \
  https://localhost:9000/userinfo \
  -H 'authorization: Bearer SwMfFOSHEFpiChmBvRtFLTeaPzCh-TNEXtxTfibgmdw.AgqJrWyn1VlH4FouUucBJSDsmcwOGDI3cHpuy2sDrpI' \
  -H 'cache-control: no-cache' \
  -H 'content-type: application/json' \
  -H 'postman-token: fecb7032-db0a-bb1b-a61b-de22add7e5bc'
```

用户信息如下:

```json
{
    "sub": "foo@bar.com"
}
```

为什么这里用户信息只有一个sub呢?

因为他们实现ory-hydra-example--consent的时候什么都没加进去,

具体应该怎么做等我下次分享dotnet core实现 login-consent再说了..

嗯,本文结束.