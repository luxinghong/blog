---
layout: post
title: "记一次 centos7 + vue + nginx + uwsgi + django-rest + mysql 的部署过程"
subtitle: ""
author: ""
header-style: text
tags:
  - python
---



弄完了感觉也没那么复杂，但的确花了我2天的时间，**主要是因为python版本和虚拟环境的问题**。

**关键的关键是要在虚拟环境中启动uwsgi，它才能找到各种依赖包。**



# 安装python3

因为centos7预装的是python2，但现在的程序都用的是pyhton3，所以要装上。

下载python3.7.4源码包： 

```bash
wget https://www.python.org/ftp/python/3.7.4/Python-3.7.4.tgz
```

 解压缩：

```bash
tar -zxvf Python-3.7.4.tgz
```

 进入解压后的目录，执行下面命令手动编译：

```bash
./configure prefix=/usr/local/python3 
make && make install
```

添加软链接： 

```bash
ln -s /usr/local/python3/bin/python3.7 /usr/bin/python3
```

python3自带了pip3，所以同样为pip3添加软链接：

```bash
ln -s /usr/local/python3/bin/pip3.7 /usr/bin/pip3
```

最后python3 -V 和 pip3 -V检查版本对不对

这样的话系统中就存在了两个版本的python和pip，想用哪个就用哪个。





# 安装uwsgi

注意得使用pip3安装，这样对应的版本才是python3。

执行命令 pip3 install uwsgi ，会被安装到python3对应的lib/site-packages目录下。

uwsgi --version 检查版本。





# 创建虚拟环境

这里我没有使用 `virtualenv` 命令，直接使用 `python3 -m venv env` 命令。

从django项目中导出requirement.txt。

激活虚拟环境：

```bash
source env/bin/activate
```

安装依赖：

```bash
pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple -r requirement.txt。
```







#  配置uwsgi配置文件

```bash
vim uwsgi.ini
```



```
socket = 127.0.0.1:3031
chdir = /home/head/uwb/
wsgi-file = uwb/wsgi.py
home = /home/head/env/
processes = 4
threads = 2
stats = 127.0.0.1:9191
```

**home 指向刚刚创建的虚拟环境目录，这很重要，否则会报各种找不到module的错误**。其他配置项就不解释了，自行查阅。





#  配置nginx

安装过程就不讲了。

首先配置前端代码。在Vue项目中使用 `npm run build` 命令生成dist目录，上传至nginx目录，添加如下配置：

```yaml
 location / {
            root   dist;
            index  index.html;
        }
```



接下来配置接口访问，也就是访问django应用。为了与静态资源区分开，将所有的接口访问都加上了前缀/api。我们不需改动我们的后端django代码，在nginx和uwsgi中配置可以达到这一效果。配置如下：

```yaml
 location /api {
        uwsgi_param SCRIPT_NAME /api;
        include  uwsgi_params;
        uwsgi_pass 127.0.0.1:3031;
    }
```

   

这里 uwsgi_pass 和 uwsgi.ini 配置文件中的socket配置项一致，这就是nginx和uwsgi通讯的端口。

同时在uwsgi.ini 配置文件中加入一行：`route-run = fixpathinfo:` ， 注意最后是冒号，表示传过来的请求都有一个固定前缀。





 

# 启动uwsgi

当然要先启动nginx、mysql。

**最好还是在虚拟环境中启动，避免出现一些莫名其妙的import error。**

执行命令：

```bash
uwsgi uwsgi.ini
```



**如果出现 `mysql_config not found` ，再安装以下东西**：

```bash
sudo yum install mysql-devel gcc gcc-devel python-devel
sudo easy_install mysql-python
```



OK，大功告成！

