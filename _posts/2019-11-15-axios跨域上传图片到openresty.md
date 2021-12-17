---
layout: post
title: "axios跨域上传图片到openresty"
subtitle: ""
author: ""
header-style: text
tags:
  - nginx
---



# 什么是**跨域**

同源：当协议、域名、端口一致的时候2个域名是同源的。比如 `aaa.cn` 和 `aaa.cn/xx/xxx` 是同源的，但和 `bbb.cn` 就不是同源的。

 



# 为什么要设置**同源策略限制**？

假如你登陆了一个银行网站，在没有退出的情况下登录了另一个网站，这个网站悄悄的携带了你银行网站的cookie对银行网站发起了请求，这样就能冒充你做所有的操作，因为银行网站并不知道到底是不是你发起的请求，它看到了你的cookie就认为是你了。所以要有这个限制。这也是CSRF(cross site request forgery) 跨站请求伪造攻击。

 



# CORS(cross origin resource sharing) 跨域资源共享

这是一个w3c标准，用于控制跨域请求。它定义了一些headers：

- Access-Control-Allow-Origin


​		响应首部中可以携带这个头部表示服务器允许哪些域可以访问该资源，其语法如下：

​		`Access-Control-Allow-Origin: <origin> | *`
​		其中，origin 参数的值指定了允许访问该资源的外域 URI。对于不需要携带身份凭证的请求，服务器可以指定该字段的值为通配符，表示允许来自所有域的请求。

<br/>

- Access-Control-Allow-Methods


​		该首部字段用于预检请求的响应，指明实际请求所允许使用的HTTP方法。其语法如下：

​		`Access-Control-Allow-Methods: <method>[, <method>]*`

<br/>

- Access-Control-Allow-Headers

​		该首部字段用于预检请求的响应。指明了实际请求中允许携带的首部字段。其语法如下：

​		`Access-Control-Allow-Headers: <field-name>[, <field-name>]*`
 	  我们就可以控制服务器上哪些资源可以被什么样的跨域请求访问，以达到跨域资源共享的目的。



接下来进入代码部分

axios部分：

```javascript
var formData = new FormData()
      var { action, onError, onSuccess, onProgress, filename, file } = data
      formData.append(filename, file)
      originalAxios.post(action, formData, {
        onUploadProgress: ({ total, loaded }) => {
          // 上传进度输出
          onProgress({ percent: Number(Math.round(loaded / total * 100).toFixed(2)) }, file)
        }
      }).then(res => {
        // 上传成功，触发handleChange。file的status会被设为done，同时将res.data赋给file.response
        onSuccess(res.data, file)
      }).catch(onError)
```

这里用到了formData，axios的onUploadProgress(上传进度)。

问题是在发post请求之前浏览器会自动先发一个options请求，用于确定服务器是否能接受这个请求，可以的话再发第二个真正的post请求。但是openresty中没有处理options请求的逻辑，会导致请求失败。所以需要在配置文件中加入以下这段代码：

 ```
 location /upload {
            if ($request_method = OPTIONS){
               add_header Access-Control-Allow-Origin *;
               add_header Access-Control-Allow-Method POST;
               return 200;
            }
            add_header Access-Control-Allow-Origin *;
            default_type 'application/json';
            content_by_lua_file lua/myupload.lua;
         }
 ```

这里对options请求返回的响应头中加入了以上提到过的COSR头部：Access-Control-Allow-Origin、Access-Control-Allow-Method，然后返回200。在第二个post请求中用lua-resty-upload来真正处理上传，这里就不细表。