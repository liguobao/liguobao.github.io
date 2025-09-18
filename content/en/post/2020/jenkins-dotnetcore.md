---
layout: post
title: CI/CD Auto-Deploy .NET Core with Jenkins + Docker
category: dotnet core
date: 2018-05-08
tags:
- dotnet core
- Jenkins
- docker
---



Install JDK 8, then run Jenkins from the war:

```sh
sudo add-apt-repository ppa:webupd8team/java
sudo apt-get update
sudo apt-get install oracle-java8-installer

wget http://mirrors.jenkins.io/war-stable/2.107.2/jenkins.war
mkdir ~/jenkins-home && export JENKINS_HOME=~/jenkins-home
tmux
java -jar jenkins.war
```

Open port 8080, use the initial admin password from `.../secrets/initialAdminPassword`, install suggested plugins, create admin.

We’ll return to Jenkins after preparing Docker builds for our .NET Core app.

## Dockerfile for .NET Core

In your project folder, `Dockerfile`:

```dockerfile
FROM microsoft/aspnetcore-build:2.0 AS build-env
WORKDIR /app
COPY *.csproj ./
RUN dotnet restore
COPY . ./
RUN dotnet publish -c Release -o out

FROM microsoft/aspnetcore:2.0
WORKDIR /app
COPY --from=build-env /app/out .
ENTRYPOINT ["dotnet", "YourApp.dll"]
```

## Optional local build/run script

```sh
image_version=$(date +%Y%m%d%H%M)
docker stop house-web || true && docker rm house-web || true
docker build -t house-web:$image_version .
docker run -p 8080:80 \
  -v ~/docker-data/house-web/appsettings.json:/app/appsettings.json \
  -v ~/docker-data/house-web/NLogFile/:/app/NLogFile \
  --restart=always --name house-web -d house-web:$image_version
```

## Jenkins Job

- New item → Freestyle project
- Source Code Management → Git (configure credentials/keys as needed)
- Build → “Execute shell” with either the local build script above, or a remote `ssh user@host '~/start_XXX.sh'`.

## Better builds with Aliyun Container Registry (ACR)

Instead of building on Jenkins/host, connect your repo to Aliyun Container Registry (ACR) and enable “auto build on code changes”. ACR will build and host your images.

Then, use a webhook from ACR to trigger Jenkins after a successful build.

### Generic Webhook Trigger

Install the “Generic Webhook Trigger” plugin. The trigger URL looks like:

```
http://<user>:<apiToken>@<jenkins-host>:8080/generic-webhook-trigger/invoke?token=<job-token>
```

Set the same token under the job’s “Trigger builds remotely”. Disable CSRF protection if needed for webhook calls.

Flow:

repo change → ACR auto build → ACR webhook → Jenkins job → deployment script pulls and runs the new image.

Deployment step becomes:

```sh
docker stop house-web || true && docker rm house-web || true
docker pull <your-acr-image>
docker run --restart=always --name house-web <your-acr-image>
```

## Summary

1) Stand up Jenkins. 2) Add Dockerfile. 3) Use ACR to build images. 4) Use ACR webhook → Jenkins to deploy. 5) Works for other stacks too — just change the Dockerfile.

