---
title: 浅显易懂的Git
date: 2016-8-22
tags:
  - 
  - 
comments: true
categories: Git
---

> 本文只是简单的记录了自己开发中涉及到的命令,后期也会不断更新.
[廖雪峰Git教程](http://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000)
[或许是介绍Android Studio使用Git最详细的文章](http://wensibo.top/2017/03/12/GitOnAS/)

## 安装Git
- 在Linux上安装Git
如果你碰巧用Debian或Ubuntu，通过一条`sudo apt-get install git`就可以直接完成Git的安装，非常简单。
- Windows安装git
[下载Windows版本Git](https://git-scm.com/download/win)
安装完成后，在开始菜单里找到`“Git”->“Git Bash”`，蹦出一个类似命令行窗口的东西，就说明Git安装成功！
![Git Bash](https://upload-images.jianshu.io/upload_images/3846387-8b30a6b7d3b1d33f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 初次运行 Git 前的配置
一般在新的系统上，我们都需要先配置下自己的 Git 工作环境。配置工作只需一次，以后升级时还会沿用现在的配置。当然，如果需要，你随时可以用相同的命令修改已有的配置。
### 用户信息
当安装完 Git 应该做的第一件事就是设置你的用户名称与邮件地址。 这样做很重要，因为每一个 Git 的提交都会
使用这些信息，并且它会写入到你的每一次提交中，不可更改：
- 用户名
你需要告诉git你的名字，这个名字会出现在你的提交记录中。
```
git config --global user.name "Scott Chacon"
```
- Email
然后是就是你的Email，同样，这个Email也会出现在你的提交记录中，这个Email可以和注册的邮箱一样,也可以用原有项目的企业邮箱等。
```
git config --global user.email "schacon@gmail.com"
```
这两条配置很重要，每次 Git 提交时都会引用这两条信息，说明是谁提交了更新，所以会随更新内容一起被永久纳入历史记录：
注意git config命令的--global参数，用了这个参数，表示你这台机器上所有的Git仓库都会使用这个配置，当然也可以对某个仓库指定不同的用户名和Email地址。
如果用了 --global 选项，那么更改的配置文件就是位于你用户主目录下，以后你所有的项目都会默认使用这里配置的用户信息。如果要在某个特定的项目中使用其他名字或者电邮，只要去掉 --global 选项重新配置即可，新的设定保存在当前项目的 .git/config 文件里。

### 查看配置信息
要检查已有的配置信息，可以使用 git config --list 命令：
```
$ git config --list
    user.name=Scott Chacon
    user.email=schacon@gmail.com
    color.status=auto
    color.branch=auto
    color.interactive=auto
    color.diff=auto
    ...
```
Git是分布式的代码管理工具，远程的代码管理是基于SSH的，所以要使用远程的Git则需要SSH的配置.

### 配置密钥
- 查看密钥
是否已经有了ssh密钥：
```
cd ~/.ssh
```
如果没有密钥则不会有此文件夹
- 生成密钥
```
$ ssh-keygen -t rsa -C “schacon@gmail.com”
```
按3个回车，密码为空就可以。
```
//下面这两句后面跟着的路径就是生产秘钥的路径
Your identification has been saved in /c/Users/user/.ssh/id_rsa.
Your public key has been saved in /c/Users/user/.ssh/id_rsa.pub.
The key fingerprint is:
………………
最后得到了两个文件：id_rsa和id_rsa.pub
```
将公钥放配置到github上或者其他的平台,这里多说下(克隆用SSH地址,不要用https,否则会总输入用户名和密码)
![QQ截图20170324102831.jpg](http://upload-images.jianshu.io/upload_images/3846387-30655181799f85ab.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/550)
- 测试是否成功
```
$ ssh -T git@github.com
```
![QQ截图20170324102142.jpg](http://upload-images.jianshu.io/upload_images/3846387-4686486adf75e583.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/550)

### 多账号ssh配置
- 生成指定名字的密钥
```
$ ssh-keygen -t rsa -C "邮箱地址" -f ~/.ssh/github_jslite
```
会在.ssh文件夹中生成github_jslite和github_jslite.pub这两个文件
![查看](http://upload-images.jianshu.io/upload_images/3846387-49893a6da473d726.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/550)
- 密钥复制到托管平台上
```
//查看秘钥
vi ~/.ssh/github_jslite.pub
```
打开公钥文件github_jslite.pub，并把内容复制至代码托管平台上,不要多复制空格和回车
- 添加config文件
在.ssh文件夹下创建config
```
Host jslite.github.com
HostName github.com
User git
IdentityFile ~/.ssh/github_jslite
Host abc.github.com
HostName github.com
User git
IdentityFile ~/.ssh/github_abc
```
- 测试
```
ssh -T git@jslite.github.com
//@后面跟上定义的Host
例如测试码云
ssh -T git@git.oschina.net
```

### Git同步到2个以上远程库
如果你想把本地的仓库连接到某个远程服务器，你可以使用如下命令添加:
```
git remote add origin https://github.com/JSLite/JSLite.git
```
此处要注意,远程库中的文件不能比本地多(初始化远程仓库的时候不要选择任何的初始化文件,如果有也要手动删除),否则会失败,如果是要将本地的项目推送到一个新的平台也可以选择强推`git push -f`这样你就能够将你的改动推送到所添加的服务器上去了.
- 但是你可能想要把你的本地的git库，既push到github上，又push到开源中国的Git@OSC上，怎么解决呢? 
方法如下：
此处假设你已经在github上clone了你的项目, 
其次你现在想要增加一个码云远程库地址 :
`git remote set-url --add origin <url1>` 
`<url1>`就是git地址例如:`git@git.oschina.net:hqx/tst.git`
然后你还可以增加第二个地址
` git remote set-url --add origin <url2> `
增加第三个地址 
`git remote set-url --add origin <url3> `
….依次类推
添加完这些之后第一次提交前要执行:
`git push --force origin master`
进行强制提交一次,做到以上git地址远程库的统一
以后再更改就可以按部就班的执行下面的步骤提交代码了
```
git status   //查看状态
git add .    //添加所有文件
git commit -m "提交备注"  //提交到HEAD区
git push origin master //同步到远程库
```
这样就完成了添加多个地址到origin库中了,至于pull的时候下面讲解.

### 添加多个远程地址的解析
- 原理解析
`git remote set-url --add origin <url>`
就是往当前git项目的config文件里增加一行记录 
打开config文件：
![](http://upload-images.jianshu.io/upload_images/3846387-a85db8f6c5113e6a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
`git remote -v`:显示当前所有远程库的详细信息，显示格式为远程库名字 url连接(类型)
![](http://upload-images.jianshu.io/upload_images/3846387-c576e9cf3c8f418e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
所以说，你直接在config里面直接添加url来修改也是可以的，不必去执行git命令。
**注意**
使用`git push origin master`时，你可以`push到origin`的多个url地址， 
但是使用 `git pull`时，只能拉取origin里的一个url地址(即fetch-url，如上图)，这个fetch-url默认为你添加的到origin的第一个地址， 
如果你想更改，只需要更改config文件里，那三个url的顺序即可，fetch-url会直接对应排行第一的那个url连接。调整好可以用`git remote -v`看一下

### 克隆远程仓库
- 创建本地仓库,在文件夹中执行:
```
git init
```
用`ls -a`命令查看目录下隐藏的`.git`文件夹。
- 创建本地仓库的克隆版本：
```
git clone /path/to/repository 
```
- 克隆远端服务器上的仓库
```
git clone username@host:/path/to/repository
```

### 添加和提交
- 修改文件前执行
```
git status   获取当前仓库的状态
```
- 查看修改情况
```
git diff filename
```
要随时掌握工作区的状态，使用`git status`命令。
如果`git status`告诉你有文件被修改过，用`git diff`可以查看修改内容。
- 添加更改代码（把它们添加到暂存区）
```
git add <filename>
#添加文件,可以连续添加
git add <filename1>
#添加所有文件,要提前配置好忽略文件否则都会添加
git add . 
```
- 提交到 HEAD
```
git commit -m "代码提交描述"
```
现在，你的改动已经提交到了 HEAD，但是还没到你的远端仓库。
- 推送改动
你的改动现在已经在本地仓库的 HEAD 中了。执行如下命令以将这些改动提交到远端仓库：
```
git push origin master
```
可以把 master 换成你想要推送的任何分支。
记录每次更新到仓库
![image.png](https://upload-images.jianshu.io/upload_images/3846387-f1b68fdf511e6c20.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 忽略文件
在添加到暂存区前要先配置忽略文件,如果忘了配置了也没关系在没有提交到服务器之前都可以回退
GitHub 有一个十分详细的针对数十种项目及语言的 .gitignore 文件列表，你可以在
[.gitignore 文件列表](https://github.com/github/gitignore) 找到它.

### 移除文件
要从 Git 中移除某个文件，就必须要从已跟踪文件清单中移除（确切地说，是从暂存区域移除），然后提交。
可以用 git rm 命令完成此项工作，并连带从工作目录中删除指定的文件，这样以后就不会出现在未跟踪文件清
单中了。
如果只是简单地从工作目录中手工删除文件，运行 git status 时就会在 “Changes not staged for
commit” 部分（也就是 未暂存清单）看到：
```
$ rm PROJECTS.md
$ git status
On branch master
Your branch is up-to-date with 'origin/master'.
Changes not staged for commit:
(use "git add/rm <file>..." to update what will be committed)
(use "git checkout -- <file>..." to discard changes in working
directory)
deleted: PROJECTS.md
no changes added to commit (use "git add" and/or "git commit -a")
```
然后再运行 git rm 记录此次移除文件的操作：
```
$ git rm PROJECTS.md
rm 'PROJECTS.md'
$ git status
On branch master
Changes to be committed:
(use "git reset HEAD <file>..." to unstage)
deleted: PROJECTS.md
```
下一次提交时，该文件就不再纳入版本管理了。 如果删除之前修改过并且已经放到暂存区域的话，则必须要用
强制删除选项 -f（译注：即 force 的首字母）。 这是一种安全特性，用于防止误删还没有添加到快照的数据，
这样的数据不能被 Git 恢复。
另外一种情况是，我们想把文件从 Git 仓库中删除（亦即从暂存区域移除），但仍然希望保留在当前工作目录
中。 换句话说，你想让文件保留在磁盘，但是并不想让 Git 继续跟踪。 当你忘记添加 .gitignore 文件，不小
心把一个很大的日志文件或一堆 .a 这样的编译生成文件添加到暂存区时，这一做法尤其有用。 为达到这一目
的，使用 --cached 选项：
```
$ git rm --cached README
```
git rm 命令后面可以列出文件或者目录的名字，也可以使用 glob 模式。 比方说：
```
$ git rm log/\*.log
```
注意到星号 * 之前的反斜杠 \， 因为 Git 有它自己的文件模式扩展匹配方式，所以我们不用 shell 来帮忙展开。
此命令删除 log/ 目录下扩展名为 .log 的所有文件。 类似的比如：
```
$ git rm \*~
```
该命令为删除以 ~ 结尾的所有文件。

### 版本回退
在没有提交到远程仓库前一切都是可以回退的
- 查看历史提交记录
```
$ git log
commit 3628164fb26d48395383f8f31179f24e0882e1e0
Author: Michael Liao <askxuefeng@gmail.com>
Date:   Tue Aug 20 15:11:49 2013 +0800
    append GPL
commit ea34578d5496d7dd233c827ed32a8cd576c5ee85
Author: Michael Liao <askxuefeng@gmail.com>
Date:   Tue Aug 20 14:53:12 2013 +0800
    add distributed
commit cb926e7ea50ad11b8f9e909c05226233bf755030
Author: Michael Liao <askxuefeng@gmail.com>
Date:   Mon Aug 19 17:51:55 2013 +0800
    wrote a readme file
```
`git log`命令显示从最近到最远的提交日志，我们可以看到3次提交，最近的一次是`append GPL`，上一次是`add distributed`，最早的一次是`wrote a readme file`。
- 如果嫌输出信息太多，看得眼花缭乱的,可以用以下命令:
```
$ git log --pretty=oneline
3628164fb26d48395383f8f31179f24e0882e1e0 append GPL
ea34578d5496d7dd233c827ed32a8cd576c5ee85 add distributed
cb926e7ea50ad11b8f9e909c05226233bf755030 wrote a readme file
```
- 回退上一个版本
我们要把当前版本“append GPL”回退到上一个版本“add distributed”，就可以使用git reset命令：
```
$ git reset --hard HEAD^
HEAD is now at ea34578 add distributed
```
在Git中,用HEAD表示当前版本,也就是最新的提交3628164...882e1e0（注意我的提交ID和你的肯定不一样），上一个版本就是HEAD^，上上一个版本就是HEAD^^，当然往上100个版本写100个^比较容易数不过来，所以写成HEAD~100。
回退完了之后要用git log查看下,当前的提交记录,再回退的话就是以当前的参考点回退
- 用commitid回退
```
$ git reset --hard 3628164
HEAD is now at 3628164 append GPL
```
版本号没必要写全，前几位就可以了，Git会自动去找。
- 当你用`$ git reset --hard HEAD^`回退到add distributed版本时，再想恢复到`append GPL`，就必须找到`append GPL`的`commit id`。Git提供了一个命令`git reflog`用来记录你的每一次命令：
```
$ git reflog
ea34578 HEAD@{0}: reset: moving to HEAD^
3628164 HEAD@{1}: commit: append GPL
ea34578 HEAD@{2}: commit: add distributed
cb926e7 HEAD@{3}: commit (initial): wrote a readme file
```
第二行显示append GPL的commit id是3628164,所以你可以回退到这个记录上来了
```
1,HEAD指向的版本就是当前版本，
Git允许我们在版本的历史之间穿梭，使用命令git reset --hard commit_id。
2,穿梭前，用git log可以查看提交历史，以便确定要回退到哪个版本。
3,要重返未来，用git reflog查看命令历史，以便确定要回到未来的哪个版本。
```

### 删除文件
在Git中，删除也是一个修改操作，比如先添加一个新文件test.txt到Git并且提交：
```
$ git add test.txt
$ git commit -m "add test.txt"
[master 94cdc44] add test.txt
 1 file changed, 1 insertion(+)
 create mode 100644 test.txt
```
一般情况下，你通常直接在本地目录中把没用的文件删了，或者用rm命令删了：
`$ rm test.txt`
这个时候，Git知道你删除了文件，因此，工作区和版本库就不一致了，`git status`命令会立刻告诉你哪些文件被删除了：
```
$ git status
# On branch master
# Changes not staged for commit:
#   (use "git add/rm <file>..." to update what will be committed)
#   (use "git checkout -- <file>..." to discard changes in working directory)
#
#       deleted:    test.txt
#
no changes added to commit (use "git add" and/or "git commit -a")
```

现在你有两个选择，一是确实要从版本库中删除该文件，那就用命令git rm删掉，并且git commit：
```
$ git rm test.txt
rm 'test.txt'
$ git commit -m "remove test.txt"
[master d17efd8] remove test.txt
 1 file changed, 1 deletion(-)
 delete mode 100644 test.txt
```
现在，文件就从版本库中被删除了。

另一种情况是删错了，因为版本库里还有呢，所以可以很轻松地把误删的文件恢复到最新版本：
`$ git checkout -- test.txt`
`git checkout`其实是用版本库里的版本替换工作区的版本，无论工作区是修改还是删除，都可以“一键还原”。

### git各区介绍
- 工作区（Working Directory）就是你在电脑里能看到的目录(某个文件夹)
- 版本库（Repository）工作区有一个隐藏目录.git，这个不算工作区，而是Git的版本库。
Git的版本库里存了很多东西，其中最重要的就是称为stage（或者叫index）的暂存区，还有Git为我们自动创建的第一个分支master，以及指向master的一个指针叫HEAD。

![](http://upload-images.jianshu.io/upload_images/3846387-8cf5781ce75be78b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

第一步是用git add把文件添加进去，实际上就是把文件修改添加到暂存区；
第二步是用git commit提交更改，实际上就是把暂存区的所有内容提交到当前分支。
因为我们创建Git版本库时，Git自动为我们创建了唯一一个master分支，所以，现在，git commit就是往master分支上提交更改。
你可以简单理解为，需要提交的文件修改通通放到暂存区，然后，一次性提交暂存区的所有修改。
每次修改，如果不add到暂存区，那就不会加入到commit中。
### 撤销修改
- 场景1：当你改乱了工作区某个文件的内容，想直接丢弃工作区的修改时，用命令`git checkout -- file`
- 场景2：当你不但改乱了工作区某个文件的内容，还添加到了暂存区时，想丢弃修改，分两步，第一步用命令`git reset HEAD file`
，就回到了场景1，第二步按场景1操作。
- 场景3：已经提交了不合适的修改到版本库时，想要撤销本次提交，参考版本回退，不过前提是没有推送到远程库。

### 删除文件
如果一个文件已经被提交到版本库，那么你永远不用担心误删，但是要小心，你只能恢复文件到最新版本，你会丢失最近一次提交后你修改的内容。

## 分支

- 分支图
![](http://upload-images.jianshu.io/upload_images/3846387-88817c76b6337811.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/550)
- 查看分支：git branch
```
$ git branch
* master
```
默认就一个主分支
- 创建分支：git branch <name>
当需要开发一个版本的时候需要创建一个分支
```
git branch 1.0
```
查看分支
```
$ git branch
  1.0
* master
```
- 切换分支：git checkout <name>
```
$ git checkout 1.0
Switched to branch '1.0'
```
查看分支
```
$ git branch
* 1.0
  master
```
- 创建+切换分支：git checkout -b <name>
- 合并某分支到主分支 ：git merge <某分支name>(首先确定在主分支上)
先切换到主分支
```
$ git checkout master
$ git merge 1.0
Updating 05f8852..1634a60
Fast-forward
 1.0.txt | 1 +
 1 file changed, 1 insertion(+)
 create mode 100644 1.0.txt
```
- 删除分支：git branch -d <name>
```
$ git branch -d 1.0
Deleted branch 1.0 (was 1634a60).

$ git branch
* master
```
- 将本地分支推送到远程
```
git push origin <branch>
```
除非你将分支推送到远端仓库，不然该分支就是 不为他人所见的
- 删除远程分支
假设你已经通过远程分支做完所有的工作了 也就是说你和你的协作者已经完成了一个特性并且将其合并到了
远程仓库的 master 分支（或任何其他稳定代码分支）。 可以运行带有 --delete 选项的 git push 命令来删
除一个远程分支。 如果想要从服务器上删除 1.0 分支，运行下面的命令：
```
$ git push origin --delete 1.0
To github.com:lqxue/GitTest.git
 - [deleted]         1.0
```
- 拉取
当 git fetch 命令从服务器上抓取本地没有的数据时，它并不会修改工作目录中的内容。 它只会获取数据然
后让你自己合并。 然而，有一个命令叫作 git pull 在大多数情况下它的含义是一个 git fetch 紧接着一个
git merge 命令。 如果有一个像之前章节中演示的设置好的跟踪分支，不管它是显式地设置还是通过 clone
或 checkout 命令为你创建的，git pull 都会查找当前分支所跟踪的服务器与分支，从服务器上抓取数据然
后尝试合并入那个远程分支。
由于 git pull 的魔法经常令人困惑所以通常单独显式地使用 fetch 与 merge 命令会更好一些。