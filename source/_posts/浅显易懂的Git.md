---
title: 浅显易懂的Git
date: 2016-8-22
tags:
  - Git
categories: Git
---

> 本文只是简单的记录了自己开发中涉及到的命令,后期也会不断更新.
[廖雪峰Git教程](http://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000)
[或许是介绍Android Studio使用Git最详细的文章](http://wensibo.top/2017/03/12/GitOnAS/)

## 安装Git

- 在Linux上安装Git

如果你碰巧用Debian或Ubuntu，通过一条`sudo apt-get install git`就可以直接完成Git的安装，非常简单。

<!-- more -->

- Windows安装git

[下载Windows版本Git](https://git-scm.com/download/win)
安装完成后，在开始菜单里找到`Git -> Git Bash`，蹦出一个类似命令行窗口的东西，就说明Git安装成功！

![Git Bash](https://raw.githubusercontent.com/lqxue/picture_list/master/image/GitBash_1099779f-d52c-45f5-9903-9f3a1ab7b872.png)

## 配置信息

当安装完`Git`应该做的第一件事就是设置你的用户名称与邮件地址。

- 用户名

```java
git config --global user.name "Scott Chacon"
```

- Email

```java
git config --global user.email "schacon@gmail.com"
```

这两条配置很重要，每次`Git`提交时都会引用这两条信息，说明是谁提交了更新，所以会随更新内容一起被记录进历史提交记录中。

注意`git config`命令的`--global`参数，用了这个参数，表示你这台机器上所有的Git仓库都会使用这个配置信息，如果要在某个特定的项目中使用其他名字或者邮箱，只要去掉`--global`选项在需要的项目重新配置即可，新的配置信息保存在当前项目的 `.git/config` 文件里。

### 查看配置信息

要检查已有的配置信息，可以使用 `git config --list` 命令：

```java
$ git config --list
    user.name=Scott Chacon
    user.email=schacon@gmail.com
    color.status=auto
    color.branch=auto
    color.interactive=auto
    color.diff=auto
    ...
```

### 配置密钥

Git是分布式的代码管理工具，远程的代码管理是基于SSH的，所以要使用远程的Git则需要SSH的配置.

- 查看密钥

先确定是否已经有了ssh密钥

```java
cd ~/.ssh
```

如果没有密钥则不会有此文件夹

- 生成密钥

```java
$ ssh-keygen -t rsa -C “schacon@gmail.com”
```

按3个回车，密码为空就可以。

```java
//下面这两句后面跟着的路径就是生产秘钥的路径
Your identification has been saved in /c/Users/user/.ssh/id_rsa.
Your public key has been saved in /c/Users/user/.ssh/id_rsa.pub.
The key fingerprint is:
………………
最后得到了两个文件(私钥和公钥)：id_rsa和id_rsa.pub
```

将公钥放配置到`github`上或者其他的平台,就可以使用`ssh`进行提交或者克隆代码了

![](https://raw.githubusercontent.com/lqxue/picture_list/master/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20190708165505.png)

- 测试是否成功

配置好公钥要测试下是否能连接成功

```java
$ ssh -T git@github.com
```

### 多账号ssh配置

- 生成指定名字的密钥

```java
$ ssh-keygen -t rsa -C "邮箱地址" -f ~/.ssh/github_jslite
```

会在.ssh文件夹中生成`github_jslite`和`github_jslite.pub`这两个文件

- 将公钥复制到托管平台上

打开公钥文件`github_jslite.pub`，并把内容复制至代码托管平台上,不要多复制空格和回车

- 添加config文件

在.ssh文件夹下创建`config`文件,文件没有后缀名,把下面的内容复制进去

```java
Host jslite.github.com
HostName github.com
User git
IdentityFile ~/.ssh/github_jslite
Host abc.github.com
HostName github.com
User git
IdentityFile ~/.ssh/github_abc
```

- 测试新添加的`git`地址

测试新添加的`git`地址,当然此处可以放到码云上,然后进行测试

```java
ssh -T git@jslite.github.com
```

## Git同步

### 同步到远程库

如果你想把本地的仓库连接到某个远程服务器，你可以使用如下命令添加远程地址:

```java
git remote add origin git@jslite.github.com
```

### 同步到2个以上远程库

此处假设你已经在github上clone了你的项目,其次你现在想要增加一个码云远程库地址 :

`git remote set-url --add origin <url1>`

`<url1>`就是git地址,可以执行多条语句添加多个`url`

例如:
`git@git.oschina.net:hqx/tst.git`
然后你还可以增加第二个地址
` git remote set-url --add origin <url2> `
增加第三个地址
`git remote set-url --add origin <url3> `
….依次类推
添加完这些之后第一次提交前要强制提交:
`git push --force origin master`

进行强制提交一次,做到以上git地址远程库的统一

如果不想强制提交的话,那就要保证远程库中的文件不能比本地多(初始化远程仓库的时候不要选择任何的初始化文件,如果有也要手动删除),否则会失败.

以后再更改就可以按部就班的执行下面的步骤提交代码了

```java
git status   //查看状态
git add .    //添加所有文件
git commit -m "提交备注"  //提交到HEAD区
git push origin master //同步到远程库
```

这样就完成了添加多个地址到origin库中了,至于pull的时候下面讲解.

### 添加多个远程地址的解析

- 原理解析

`git remote set-url --add origin <url>`

就是往当前git项目的`config`文件里增加一行记录,打开`config`文件：

```java
[core]
  repositoryformatversion = 0
  filemode = false
  bare = false
  logallrefupdates = true
  symlinks = false
  ignorecase = true
  hideDotFiles = dotGitOnly
[remote "origin"]
  url = git@10.12.133.21:andxxd/xxx.git
  fetch = +refs/heads/*:refs/remotes/origin/*
  url = git@github.com:xxx/xxx.git
[branch "master"]
  remote = origin
  merge = refs/heads/master
```

所以说，你直接在config里面直接添加url来修改也是可以的，不必去执行git命令。

最后可以通过`git remote -v`:显示当前所有远程库的详细信息

```java
$ git remote -v
origin  git@10.12.133.21:andxxd/xxx.git (fetch)
origin  git@10.12.133.21:andxxd/xxx.git (push)
origin  git@github.com:xxx/xxx.git (push)
```

**注意**

使用`git push origin master`时，你可以`push`到远程的多个url地址，但是使用 `git pull`时，只能拉取`fetch-url`远程仓库的内容，这个fetch-url默认是你`config`文件中的第一个地址，如果你想更改，只需要更改`config`文件里那三个url的顺序即可，fetch-url会直接对应排行第一的那个url连接。调整好可以用`git remote -v`看一下

```java
$ git remote -v
origin  git@10.12.133.21:andxxd/xxx.git (fetch)
origin  git@10.12.133.21:andxxd/xxx.git (push)
origin  git@github.com:xxx/xxx.git (push)
```

## 克隆远程仓库

- 克隆远端服务器上的仓库

```java
git clone git@github.com:xxx/xxx.git
```

## 添加和提交

- 修改文件前执行

```java
git status   获取当前仓库的状态
```

- 查看修改情况

```java
git diff filename
```

要随时掌握工作区的状态，使用`git status`命令。
如果`git status`告诉你有文件被修改过，用`git diff`可以查看修改内容。

- 添加更改代码（把它们添加到暂存区）

```java
#添加某个文件
git add <filename>
```

```java
#添加所有文件,要提前配置好忽略文件否则都会添加
git add .
```

- 提交到 HEAD

```java
git commit -m "代码提交描述"
```

现在，你的改动已经提交到了 HEAD，但是还没到你的远端仓库。

- 推送到远程仓库

你的改动现在已经在本地仓库中了。执行如下命令以将这些改动提交到远端仓库(主分支)：

```java
git push origin master
```

![](https://raw.githubusercontent.com/lqxue/picture_list/master/image/3846387-f1b68fdf511e6c20.png)

## 忽略文件

在添加到暂存区前要先配置忽略文件,避免提交多余的大文件,`GitHub`有一个十分详细的针对数十种项目及语言的`.gitignore`文件列表，你可以在
[.gitignore 文件列表](https://github.com/github/gitignore) 找到参考它.

## 移除文件

### 仅仅删除暂存区里的文件

此时你想撤销错误添加到暂存区里的文件，可以输入以下命令：

`git rm --cache 文件名`

上面的命令仅仅删除暂存区的文件而已，不会影响工作区的文件.

### 删除暂存区和工作区的文件

`git rm -f 文件名`

暂存区和工作区的文件都被删除了。

### 删错了

如果是删错了，因为版本库里还有呢，所以可以很轻松地把误删的文件恢复到最新版本：

`git checkout -- test.txt`

其实是用版本库里的版本替换工作区的版本，无论工作区是修改还是删除，都可以“一键还原”。

### 删除错误提交的commit

有时，不仅添加到了暂存区，而且commit到了版本库，这个时候就不能使用git rm了，需要使用git reset命令。

错误提交到了版本库，此时无论工作区、暂存区，还是版本库，这三者的内容都是一样的，所以在这种情况下，只是删除了工作区和暂存区的文件，下一次用该版本库回滚那个误添加的文件还会重新生成。

这个时候，我们必须撤销版本库的修改才能解决问题！

`git reset`有三个选项，`--hard`、`--mixed`、`--soft`。

```java
//仅仅只是撤销已提交的版本库，不会修改暂存区和工作区
git reset --soft 版本库ID

//仅仅只是撤销已提交的版本库和暂存区，不会修改工作区
git reset --mixed 版本库ID

//彻底将工作区、暂存区和版本库记录恢复到指定的版本库
git reset --hard 版本库ID
```

那我们到底应该用哪个选项好呢？

1. 如果你是在提交了后，对工作区的代码做了修改，并且想保留这些修改，那么可以使用`git reset --mixed 版本库ID`，注意这个版本库ID应该不是你刚刚提交的版本库ID，而是刚刚提交版本库的上一个版本库。如下`ID`应该用`ea34578d5496d7dd233c827ed32a8cd576c5ee85`

查看历史提交记录

```java
$ git log
commit 3628164fb26d48395383f8f31179f24e0882e1e0
Author: Michael Liao <askxuefeng@gmail.com>
Date:   Tue Aug 20 15:11:49 2013 +0800
    append GPL
commit ea34578d5496d7dd233c827ed32a8cd576c5ee85
Author: Michael Liao <askxuefeng@gmail.com>
Date:   Tue Aug 20 14:53:12 2013 +0800
    add distributed
```

2. 如果不想保留这些修改，可以直接使用彻底的恢复命令`git reset --hard 版本库ID`

3. 为什么不使用--soft呢，因为它只是恢复了版本库，暂存区仍然存在你错误提交的文件索引，还需要进一步使用上一节的`删除错误添加到暂存区的文件`，详细见上文。

- 如果嫌输出信息太多，看得眼花缭乱的,可以用以下命令:

```java
$ git log --pretty=oneline
3628164fb26d48395383f8f31179f24e0882e1e0 append GPL
ea34578d5496d7dd233c827ed32a8cd576c5ee85 add distributed
cb926e7ea50ad11b8f9e909c05226233bf755030 wrote a readme file
```

- 当你回退到某个版本时，再想恢复到之前的版本，就必须找到那个版本的的`commit id`。Git提供了一个命令`git reflog`用来记录你的每一次命令：

```java
$ git reflog
ea34578 HEAD@{0}: reset: moving to HEAD^
3628164 HEAD@{1}: commit: append GPL
ea34578 HEAD@{2}: commit: add distributed
cb926e7 HEAD@{3}: commit (initial): wrote a readme file
```

1. HEAD指向的版本就是当前版本，Git允许我们在版本的历史之间穿梭
2. 穿梭前，用`git log`可以查看提交历史，以便确定要回退到哪个版本。
3. 要重返未来，用`git reflog`查看命令历史，以便确定要回到未来的哪个版本。


### 撤销修改

- 场景1：当你改乱了工作区某个文件的内容，想直接丢弃工作区的修改时，用命令`git checkout -- file`

- 场景2：当你不但改乱了工作区某个文件的内容，还添加到了暂存区时，想丢弃修改，分两步，第一步用命令`git reset HEAD file`
，就回到了场景1，在执行场景1操作。

- 场景3：已经提交了不合适的修改到版本库时，想要撤销本次提交，参考版本回退，不过前提是没有推送到远程库。

- 场景4: 如果已经提交到远程仓库了,那就只能执行完上面三个步骤后使用强制推送到远程仓库了`git push -f origin master`,这个会把本地的历史记录以及文件都提交到远程了,慎用

## git各区介绍

- 工作区（Working Directory）就是你在电脑里能看到的目录(某个文件夹)

- 版本库（Repository）工作区有一个隐藏目录.git，这个不算工作区，而是Git的版本库。Git的版本库里存了很多东西，其中最重要的就是称为`stage`的暂存区，还有Git为我们自动创建的第一个分支master，以及指向master的一个指针叫HEAD。

![](https://raw.githubusercontent.com/lqxue/picture_list/master/image/3846387-8cf5781ce75be78b.jpg)

## 分支

### 分支图

![](https://raw.githubusercontent.com/lqxue/picture_list/master/image/3846387-88817c76b6337811.jpg)

### 查看分支

```java
$ git branch
* master
```

### 创建分支

```java
git branch V1.0
```

创建后查看分支,星号所在就是当前分支

```java
$ git branch
  V1.0
* master
```

### 切换分支

```java
$ git checkout V1.0
Switched to branch 'V1.0'
```

然后查看分支

```java
$ git branch
* V1.0
  master
```

### 创建+切换分支

```java
git checkout -b <name>
```

### 合并某分支到主分支 ：

`git merge <被合并的分支name>`

比如需要将`V1.0`分支合并到主分支上

先切换到主分支

```java
$ git checkout master
```

然后合并V1.0
```java
$ git merge V1.0
Updating 05f8852..1634a60
Fast-forward
 1.0.txt | 1 +
 1 file changed, 1 insertion(+)
 create mode 100644 1.0.txt
```

### 删除分支

`git branch -d <name>`

```java
$ git branch -d V1.0
Deleted branch V1.0 (was 1634a60).

$ git branch
* master
```

### 将本地分支推送到远程

```java
git push origin <branch name>
```

### 将远程分支拉取到本地

```java
git pull origin <branch name>
```

### 删除远程分支

假设你已经通过远程分支做完所有的工作了,也就是说你要将远程其他分支合并到远程仓库的`master`分支。 可以运行带有 `--delete` 选项的 `git push` 命令来删除一个远程分支。 如果想要从服务器上删除 `V1.0` 分支，运行下面的命令：

```java
$ git push origin --delete V1.0
To github.com:lxxx/GitTest.git
 - [deleted]         V1.0
```

## git fetch与git pull的区别

`FETCH_HEAD`： 是一个存储版本链接的文件(`.git/FETCH_HEAD`)，指向着目前已经从远程仓库取下来的分支的末端版本。
commit-id：每次`git commit`会产生一个commit-id，这是一个能唯一标识一个版本的序列号。在使用`git push`后，这个序列号还会同步到远程仓库。

有了以上的概念再来说说git fetch
git fetch：将远程仓库分支的最新`commit-id`记录到`.git/FETCH_HEAD`文件中
git fetch更新远程仓库到本地使用方式如下：

```java
//在本地新建一个temp分支，并将远程origin仓库的master分支代码下载到本地temp分支
git fetch origin master:tmp
//来比较本地代码与刚刚从远程下载下来的代码的区别
git diff tmp
//合并temp分支到本地的master分支
git merge tmp
//如果不想保留temp分支 可以用这步删除
git branch -d temp
```

`git pull`首先
比对本地的`FETCH_HEAD`记录与远程仓库的版本号，然后`git fetch` 获得当前指向的远程分支的后续版本的数据，然后再利用`git merge`将其与本地的当前分支合并。所以可以认为`git pull`是`git fetch`和`git merge`两个步骤的结合.

### 克隆远程多个分支

执行`git clone URL`后,默认只会克隆`master`分支,并将本地master分支和远程创库的msater分支建立关系,那么如何克隆出所有分支呢?

当克隆后执行`git branch -a`

```java
$ git branch -a
* master
  remotes/origin/HEAD -> origin/master
  remotes/origin/hexo
  remotes/origin/master
```

你会发现 除了主分支和远程分支建立了联系外,其他的分支并没有和远程建立联系

`git branch --track location_branch_name remote_branch_name`

这句是创建本地分支并并和远程分支建立关系

- location_branch_name是本地分支名
- remote_branch_name是远程分支名,一般的都带`origin`

比如要关联远程分支hexo和本地的hexo的联系

`git branch --track hexo origin/hexo`

```java
$ git branch -a
  hexo
* master
  remotes/origin/HEAD -> origin/master
  remotes/origin/hexo
  remotes/origin/master
```

最后发现本地和远程hexo分支建立联系了  接下来就是切换到`hexo`分支

`git checkout hexo`

切换后要pull下代码

```java
git fetch --all
git pull --all
```

这两句的意思就是更新所有分支上的代码到最新的状态
这两步执行后就可以克隆出所有分支了