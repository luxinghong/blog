---
layout: post
title: "Django 数据库迁移到MySQL"
subtitle: ""
author: ""
header-style: text
tags:
  - python
---



默认Django数据库采用的是sqlite3，想迁移到mysql数据库。



# 创建Mysql数据库

这没啥好说的，肯定你要先有个数据库吧。





# 更改Django settings配置，修改为使用MySQL数据库

```
DATABASES = {
    'default': {
        # 'ENGINE': 'django.db.backends.sqlite3',
        # 'NAME': os.path.join(BASE_DIR, 'db.sqlite3'),
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'xx',
        'USER': 'xx',
        'PASSWORD': 'xx',
        'HOST': 'xx',
        'PORT': 'xx'
    }
}
```

​        

# 安装mysqlclient

```bash
pip install mysqlclient
```





# 执行迁移命令

这一步会把你应用 migrations 目录下的所有迁移文件应用到新的数据库中。

```bash
python manager.py makemigrations
python manager.py migrate
```





# 导入数据

上一步只是在新的数据库中生成了表结构，还没有数据。

先把settings改为原来的配置，这样才能导出原来的数据。然后执行 `python manager.py dumpdata > data.json` 	 命令。

再把settings改为mysql，执行 `python manager.py loaddata data.json`   命令，这样就完成了数据的迁移。