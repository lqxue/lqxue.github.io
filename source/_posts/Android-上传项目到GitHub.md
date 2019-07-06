---
title: Android-上传项目到GitHub
date: 2016-08-23
tags:
categories: Git
---
> 开篇话:从决定写简书到现在也有一段时间了,平时看到的好的项目和技能点都是当时记住了,可有时候好记性不如烂笔头,当项目做的久了,知识多了之后就会迷茫想翻看之前的知识都无从下手,所以写简书就是为了做个笔记,内容不一定很难,只要是方便日后查阅就好.

## 配置公钥
- 配置公钥相关的内容可以参考[浅显易懂的Git](http://www.jianshu.com/p/588d09d48e9a)
登录GitHub选择Settings -> SSH and GPG keys
![Settings](http://upload-images.jianshu.io/upload_images/3846387-6b8b7489d006bb7f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](http://upload-images.jianshu.io/upload_images/3846387-ed889b71bbd1c864.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/3846387-d1f1b972761804ac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)

我已经添加过了所以是上面的界面,要是第一次添加就有个添加选项,公钥在.ssh文件中,copy下就可以了.

## 创建GitHub仓库
这的仓库名字可以随意取名字,大小写也随意,但是尽量和AS中一样好区分.
![创建代码库](http://upload-images.jianshu.io/upload_images/3846387-7cf5483a006f2710.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/3846387-6ad84b48b2eb6409.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在这要copy SSH git地址(因为公钥是用SSH协议生成的),此处我要说下:要是使用HTTPS的git地址那就每次提交的时候要验证用户名和密码,所以此处用SSH,这样只要公私钥匹配正确就不用每次都验证密码了.

## 创建本地项目

在AndroidStudio中创建一个Test的项目,然后找到本地文件路径,右击->Git Bash
![](http://upload-images.jianshu.io/upload_images/3846387-ce01dad494c7f7cc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
然后就是clone刚才的项目到本地
![](http://upload-images.jianshu.io/upload_images/3846387-776e5e1792c0dd4e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
成功之后将克隆下的Test文件夹中的文件copy到外层本地项目中的Test中,然后删掉clone下来的Test文件夹
![](http://upload-images.jianshu.io/upload_images/3846387-e93c95f98a40f0ca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
最后看下最后的结果:
![](http://upload-images.jianshu.io/upload_images/3846387-2b8a5ff0ec9615de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
现在就完成的差不多了,接下来就需要几个命令就完事了,其中的忽略文件自己看着写就行(网上一堆)
接下来再次回到命令行:
- `git add .`添加所有文件到暂存区
![](http://upload-images.jianshu.io/upload_images/3846387-84bd4342786173cf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- `git commit -m "描述信息"`将暂存区的文件提交到Head区
![](http://upload-images.jianshu.io/upload_images/3846387-a51141b37e83cede.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- `git push origin master`最后提交到远程GitHub主分支
![](http://upload-images.jianshu.io/upload_images/3846387-2c7c07118de427b2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 最后刷新GitHub

![](http://upload-images.jianshu.io/upload_images/3846387-b119b7a5a3f5f317.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 结束语

[GitHub地址](https://github.com/lqxue/Test)
弄完这些也算是告一段落了,虽然内容很简单,但是我坚信还有对一些人有用处,也是对自己这个知识的一个总结吧!

