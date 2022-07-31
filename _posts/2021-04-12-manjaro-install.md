---
layout: post
title: "Manjaro安装配置踩坑"
subtitle: ""
author: ""
header-style: text
tags:
  - Manjaro
---

### LVM方式的安装

我安装的版本是Manjaro 21 KDE，常规安装按图形界面方式很快，没有任何问题。但 [calamares](https://github.com/calamares/calamares)当前版本对 lvm 的支持还不是很好，直接在图形界面采用 lvm 点下一步后会马上闪退，或者进入安装过程后报各种卷创建的错误，实在无招，只能曲线救国。

最开始手动创建卷，然后进图形界面选择文件系统和挂载点，安装倒是没问题了，最后也提示安装成功。但重启后找不到系统，还是会继续进入U盘。推测是 calamares 在 lvm 方式下创建 efi 分区有bug，gitHub上也有很多相关issue。

试了N遍之后，最后的办法是先按常规方式安装一遍，保证重启后能正常进入系统。再用U盘进入live environment，在已分区的基础上手动创建物理卷，卷组和逻辑卷。efi 和 swap 不要动(注意在后续图形安装界面中 efi 千万别选择格式化，要的就是保留 efi 之前的内容)，其他分区可自由分配卷组和逻辑卷。最后进入图形安装界面选择挂载点和文件系统安装即可。





### Intel核显 + NVIDIA独显 的问题

> Linus Torvalds:  NVIDIA, fuck u!

安装完之后一开始正常使用没什么问题，但是一竖屏后就有撕裂现象。

解决办法：安装 optimus-manager 和 optimus-manager-qt ，切换为 NVIDIA 之后就没问题了。大黄蜂方案已经过时且不再维护了，不建议使用。

[optimus-manager](https://github.com/Askannz/optimus-manager) ( A Linux program to handle GPU switching on Optimus laptops)

**文档中有关于 [Gnome and GDM users](https://github.com/Askannz/optimus-manager#important--gnome-and-gdm-users) 和 [IMPORTANT : Manjaro KDE users](https://github.com/Askannz/optimus-manager#important--manjaro-kde-users) 的注意事项，一定要看，否则可能会导致切换后黑屏、无法启动的问题，我在这浪费了好多时间...**



### 换源

`sudo pacman-mirros -i -c China -m rank ` 	 选中一个

`sudo pacman -Syy`  	更新软件源

`sudo pacman -Syu` 	 更新全部软件





### 安装搜狗拼音

添加 archlinuxcn 软件源，/etc/pacman.conf 在最后加入

```[archlinuxcn] 
SigLevel = Optional TrustAll 
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch
```

`sudo pacman -Syy`  	更新软件源

`sudo pacman -S archlinux-keyring`	 更新软件密钥

`sudo pacman -S fcitx-im`	 安装输入法框架

`sudo pacman -S fcitx-configtool`	 安装输入法配置工具

在软件中心找到并安装 `fcitx-sogoupinyin`

<u>( 注意：一定要按顺序安装，不能直接安装最后一个，会导致依赖缺失，不能配置等问题)</u>

在 /etc/environment 中加入

```
GTK_IM_MODULE=fcitx
QT_IM_MODULE=fcitx
XMODIFIERS=@im=fcitx
```

重启完成







### 安装微信

文档写的很详细：[deepin-wine-wechat-arch](https://github.com/countstarlight/deepin-wine-wechat-arch)

中文字体显示为框框：

1. 下载宋体字符集文件：https://www.freefonts.io/downloads/simsun/
2. 将解压后的文件复制到如下路径：cp    SIMSUN.ttf    ~/.deepinwine/Deepin-WeChat/drive_c/windows/Fonts/
3. 重启微信





### 使用 oh-my-zsh

https://gist.github.com/yovko/becf16eecd3a1f69a4e320a95689249e

[主题美化](https://github.com/romkatv/powerlevel10k#arch-linux)







### IDEA Ultimate 输入法候选框不跟随鼠标，一直在左下角

这是一个 idea 古老又没人管的bug

解决办法参考：https://ld246.com/article/1601280084643 

下载修改过的  JetBrainsRuntime ，在 idea.sh 启动脚本开头添加

`export IDEA_JDK=xport IDEA_JDK=目录/java-11.0.7-jetbrain`

但是有个新的问题，markdown文件 的预览按钮不见了，也懒得折腾了，写 markdown 用专门的编辑器吧，比如 Typoar







###  IDEA 不会显示全局系统菜单 

Manjaro安装了 Application Title 插件后所有应用的菜单都会在顶部的全局系统菜单显示，但 IDEA 不会。

官方解决办法： idea中安装 JavaFX Runtime for Plugins 插件。

但是有点延迟，打开 idea 后要过一会菜单才会出来，也是很烦。





### 美化

[Make Your KDE Plasma Desktop Look Better](https://www.youtube.com/watch?v=exQh0_JKBJQ)

参考着来，不需要完全照搬





### pamac 安装软件超时

修改`/etc/makepkg.conf`使用自定义脚本通过镜像网站和`axel`多线程下载。

- 修改前：

  ```shell
  DLAGENTS=('file::/usr/bin/curl -qgC - -o %o %u'
             'ftp::/usr/bin/curl -qgfC - --ftp-pasv --retry 3 --retry-delay 3 -o %o %u'
             'http::/usr/bin/curl -qgb "" -fLC - --retry 3 --retry-delay 3 -o %o %u'
             'https::/usr/bin/curl -qgb "" -fLC - --retry 3 --retry-delay 3 -o %o %u'
             'rsync::/usr/bin/rsync --no-motd -z %u %o'
             'scp::/usr/bin/scp -C %u %o')
  ```

- 修改后：

  ```shell
  DLAGENTS=('file::/usr/bin/curl -qgC - -o %o %u'
             'ftp::/usr/bin/axel -n 15 -a -o %o %u'
             'http::/usr/bin/axel -n 15 -a -o %o %u'
             'https::/usr/bin/fake_curl_makepkg %o %u'
             'rsync::/usr/bin/rsync --no-motd -z %u %o'
             'scp::/usr/bin/scp -C %u %o')
  ```

脚本文件 fake_curl_makepkg 如下，放到 `/usr/bin` 下，不带 `sh后缀` ：

```shell
#! /bin/bash
# 该脚本用于处理yay安装软件时，由github下载缓慢甚至无法下载的问题
# 检测域名是不是github，如果是，则替换为镜像网站，依旧使用curl下载
# 如果不是github则采用axel代替curl进行15线程下载
# 实验用链接：
# https://download.fastgit.org/beekeeper-studio/beekeeper-studio/releases/download/v1.6.11/beekeeper-studio_1.6.11_amd64.deb
# https://github.com/beekeeper-studio/beekeeper-studio/releases/download/v1.6.11/beekeeper-studio_1.6.11_amd64.deb

domin=`echo $2 | cut -f3 -d'/'`;
others=`echo $2 | cut -f4- -d'/'`;
case "$domin" in 
    "github.com")
    url="https://download.fastgit.org/"$others;
    echo "download from github mirror $url";
    /usr/bin/curl -x socks5://127.0.0.1:1089 -gqb "" -fLC - --retry 3 --retry-delay 3 -o $1 $url;
    ;;
    *)
    url=$2;
    export https_proxy="https://127.0.0.1:8889"
    /usr/bin/axel -n 15 -a -o $1 $url;
    ;;
esac
```







### VMware Workstation



#### 第一次启动虚拟机时报错 

`Cannot open /dev/vmmon: No such file or directory. Please make sure that the kernel module vmmon' is loaded。` 原因是内核中没有`vmmon`模块。首先推荐使用LTS版本，其次较新的版本可能还没有 `vmmon`模块的补丁。执行以下命令安装 linux-headers：

  ```shell
  sudo pacman -S $(pacman -Qsq "^linux" | grep "^linux[0-9]*[-rt]*$" | awk '{print $1"-headers"}' ORS=' ')
  ```

  重启后执行：

  ```shell
  sudo vmware-modconfig --console --install-all
  ```

  [参考链接1](https://forum.manjaro.org/t/could-not-open-dev-vmmon/21431/2)

  [参考链接2](https://www.leeyiding.com/archives/43/) 　　

  

####  打开虚拟网络编辑器报错 

`Fail Network configuration is missing. Ensure that /etc/vmware/-networking exists.`

  执行以下命令启动 `vmware-networks-configuration` 服务：

  ```shell
  systemctl start vmware-networks-configuration.service
  ```





#### 虚拟机无网络

报错 `Could not connect 'Ethernet0' to virtual network '/dev/vmnet0'`。

  重装 linux-headers：

  ```shell
  sudo pacman -R linux510-headers
  
  sudo pacman -S linux510-headers
  ```

  重置网卡：

  ```shell
  sudo touch /etc/vmware/x && sudo vmware-networks --migrate-network-settings /etc/vmware/x && sudo rm /etc/vmware/x && sudo modprobe vmnet && sudo vmware-networks --start
  ```

  查看网卡状态：

  ```shell
  vmware-networks --status
  ```

  虚拟机设置里选择连接方式和网卡即可。  



  若后面再次 could not connect，执行

```shell
systemctl start vmware-networks
systemctl enable vmware-networks
```

  保证 vmware-networks 是 loaded 的就行，状态是 inactive 也没关系。

  重新在虚拟机设置里选择连接方式和网卡即可。