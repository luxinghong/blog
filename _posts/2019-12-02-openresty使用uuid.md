---
layout: post
title: "openresty使用uuid"
subtitle: ""
author: ""
header-style: text
tags:
  - nginx
---



主要是使用到了 [resty.jit-uuid](https://github.com/thibaultcha/lua-resty-jit-uuid) 这个模块，这个模块并没有集成到 OpenResty 中，可以直接从 github 上下载 jit-uuid.lua 文件，放到 OpenResty 的安装目录下的 ```lualib/resty```  目录里。