---
layout: post
title: "virtualenv环境的使用"
date:  2020-7-9 12:06:00  
categories: python
---

使用pip安装python中的第三方模块，默认的都是安装的最新版的模块，当然你也可以指定版本号,，比如`pip install requests==2.3.0`.如果在机器上开发维护的多个项目，使用的不同版本的模块，那么仅仅使用机器上提供的单一的python环境就无法解决这个问题。

那么我们可以使用虚拟环境来解决这个问题，虚拟环境相当于一个独立的空间，在这个空间里面有独立的python版本和模块版本，在独立空间里面使用pip安装的模块也是专属与空间的，多个独立的空间没有任何影响。python中常用的独立环境是`virtualenv`,下面就来说说`virtualenv`的使用.

##### 安装

`pip install virtualenv`

##### 建立虚拟环境

 `virtualenv -p C:\Python37-32\python.exe --no-site-packages  workon`

-p参数指定了使用了python版本，对于存在多个python版本的机器这个参数非常有用

--no-site-packages 不要复制机器上site-packages中的模块到虚拟环境中

这样就得到一个非常纯净的python环境workon

##### 开启环境

进入workon目录，并且在scripts目录中执行activate.bat命令，这样就进入到了虚拟环境，这个时候可以看到我们的命令提示符的前面显示的有当前的虚拟环境名称。

`(workon) e:\env\workon\Scripts>`

##### 退出环境

同样是在scripts目录下，执行deactivate.bat命令，可以退出虚拟环境



virtualenv使用上很简单，但是就是有些不太方便的地方，需要不断的切换目录来执行命令。而`virtualenvwrapper`则很好的弥补了这一点。

##### 安装

`pip install virtualenvwrapper-win`

在linux下，则是`virtualenvwrapper`,这个要注意区别

##### 建立虚拟环境

`mkvirtualenv` workon

创建一个`workon`的虚拟环境，默认是安装在 `用户名/Envs/ `目录下的。当然也可以改变默认目录，需要设置一个`WORKON_HOME`的环境变量，linux下则是这是`workon`环境变量。

##### 开启环境

`workon 环境名`

##### 关闭环境

`deactivate`

##### 查看所有环境

`lsvirtualenv`

#### 删除环境名

`rmvirtualenv 环境名`

##### linux中的安装不同

在linux中，需要配置两个环境变量

```bash
export WORKON_HOME=$HOME/.envs       #设置虚拟环境的默认路径
export VIRTUALENVWRAPPER_PYTHON=/usr/bin/python3   #设置虚拟环境的默认python版本
source /home/ubuntu/.local/bin/virtualenvwrapper.sh  #使virtualenvwrapper生效
```













