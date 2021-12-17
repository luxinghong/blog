---
layout: post
title: "dockerfile打成docker上传到linux后运行报sh脚本找不到的错误"
subtitle: ""
author: ""
header-style: text
tags:
  - docker

---



之前遇到一个奇葩错误， 本地dockerfile打成docker上传到linux后(使用的是docker-maven-plugin)，docker run imageID 一直报 `./run.sh not found ` 的错误。

dockerfile如下：

```dockerfile
FROM openjdk:8-jdk-alpine
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories
RUN apk update && apk upgrade && apk add netcat-openbsd
RUN echo $JAVA_HOME
RUN cd /tmp/ && \
    wget "http://download.oracle.com/otn-pub/java/jce/8/jce_policy-8.zip" --header 'Cookie: oraclelicense=accept-securebackup-cookie' && \
    unzip jce_policy-8.zip && \
    rm jce_policy-8.zip && \
    yes |cp -v /tmp/UnlimitedJCEPolicyJDK8/*.jar /usr/lib/jvm/java-1.8-openjdk/jre/lib/security/
ADD  @project.build.finalName@.jar /usr/local/configserver/
ADD run.sh run.sh
RUN chmod +x run.sh
CMD cd /
CMD ./run.sh
```

run.sh明明存在，为什么一直not found。

经过一番google，在stackoverflow上找到答案。文件的换行问题！！！

windows上使用的是CRLF(\r\n，回车换行)，linux上使用的是LF(\n，换行)！！！

修改方法：

![](/blog/img/20200720093335139.png)