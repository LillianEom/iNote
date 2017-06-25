---
title: Git-Workflow
toc: true
date: 2016-03-16 03:15:35
tags: git-workflow
categories:
---

## Git是什么

### Git能干什么？

[git](http://lib.csdn.net/base/git)用来做[版本控制](http://lib.csdn.net/base/git)，这就好比当初在小日本的公司，每次修改文件，都需要在文件第一页新增修改履历，详细记录修改内容，如果多人同时操作，你可以想象下维护的成本有多高。基本每天就在整理这些破事。

所以说，版本控制是项目开发的重中之重。那么问题来了，版本控制哪家强？一句话，Git是目前世界上最先进的分布式版本控制系统（没有之一）。其实也可以把分布式三个字去掉。

<!--more-->

### 集中式版本控制

集中式的版本控制工具以SVN为代表，他有一个中央服务器，控制着所有的版本管理，其它所有的终端，可以对这个中央库进行操作，中央库保证版本的唯一性。

![这里写图片描述](http://img.blog.csdn.net/20150628140633296)

这样有一个非常不好的地方，就是如果中央服务器被天灾军团攻陷了，那么整个版本就gg了。因为终端只能对一部分代码进行修改，获取各种信息，需要不断与中央服务器通信。

1. 容灾性差
2. 通信频繁

### 分布式版本控制

分布式版本控制的典型，就是Git，它跟集中式的最大区别就是它的终端可以获取到中央服务器的完整信息，就好像做了一个完整的镜像，这样，我可以在终端做各种操作、获取各种信息而不需要与服务器通信，同时，就算服务器被炸，各个终端还有完整的备份。分布式的思路，在版本控制上，具有得天独厚的优势，当然，这只是git优势的冰山一角。

## Git安装与配置

### 安装

Git的安装就不解释了，相信程序猿都有能力独立解决这样一个问题，[linux](http://lib.csdn.net/base/linux)和Mac下基本都已经自带Git了，window，建议不要用Git了，命令行实在是受不了。

### 配置

在终端输入：

```Bash
➜  ~  git --version
git version 1.8.41212
```

来查看当前Git的版本，同时，输入：

```Bash
➜  ~  git config --list --global
user.name=xuyisheng
user.email=xuyisheng@hujiang.com
push.default=current12341234
```

或者：

```Bash
➜  ~  git config user.name
xuyisheng1212
```

用来查看当前的配置信息，如果是新安装的，那么需要对global信息进行配置。配置的方法有两种，一个个变量配置，或者一起配置：

单独配置：

```Bash
➜  ~  git config --global user.name xys11
```

一起配置：

```Bash
➜  ~  git config --global --add user.name xys11
```

增加多个键值对

删除配置

```Bash
➜  ~  git config --global --unset user.name xys11
```

### 配置别名

这个功能在shell中是很常用的。我们可以做一些别名来取代比较复杂的指令。 
比如：

```Bash
git config --global alias.st status11
```

我们使用st来取代status。

附上一个比较吊的：

```Bash
git config --global alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"11
```

> PS:用git log –graph命令可以看到分支合并图。

### 配置文件

git的配置文件其实我们是可以找到的，就在.git目录下：

```Bash
➜  testGit git:(master) ls -a
.          ..         .git       README.txt
➜  testGit git:(master) cd .git
➜  .git git:(master) ls
HEAD        description index       logs        packed-refs
config      hooks       info        objects     refs
➜  .git git:(master)12345671234567
```

我们打开config文件：

```Bash
[core]
    repositoryformatversion = 0
    filemode = true
    bare = false
    logallrefupdates = true
    ignorecase = true
    precomposeunicode = false
[remote "origin"]
    url = git@github.com:xuyisheng/testGit.git
    fetch = +refs/heads/*:refs/remotes/origin/*
[branch "master"]
    remote = origin
    merge = refs/heads/master
[branch "dev"]
    remote = origin
    merge = refs/heads/dev
12345678910111213141516171234567891011121314151617
```

这里就是我们所有的配置了。

## 创建Git仓库

版本控制就是为了管理代码，代码就要放在仓库中。创建仓库有二种方法：

### Git init

```Bash
➜  MyWork  mkdir gitTest
➜  MyWork  cd gitTest
➜  gitTest  git init
Initialized empty Git repository in /Users/hj/Downloads/MyWork/gitTest/.git/
➜  gitTest git:(master)1234512345
```

创建一个目录，并cd到目录下，通过调用git init来将现有目录初始化为git仓库，或者直接在git init后面跟上目录名，同样也可以创建一个新的仓库。

### git clone

git clone用于clone一个远程仓库到本地，这个我们后面再将。

创建好仓库后，目录下会生成一个.git的隐藏文件夹，这里就是所有的版本记录，默认不要对这个文件夹进行修改。

## 提交修改

### add && commit

在仓库中，我们创建代码，并将其提交：

```Bash
➜  gitTest git:(master) touch README.txt
➜  gitTest git:(master) ✗ open README.txt
➜  gitTest git:(master) ✗ git add README.txt
➜  gitTest git:(master) ✗ git commit -m "add readme"
[master (root-commit) c19081b] add readme
 1 file changed, 1 insertion(+)
 create mode 100644 README.txt12345671234567
```

我们创建了一个README文件，并通过git add <文件名>的方式进行add操作，最后通过git commit操作进行提交，-m参数，指定了提交的说明。

这两个命令应该是最常使用的git命令。

### 查看修改

在版本控制中，非常核心的一点，就是需要指定，我们做了哪些修改，比如之前我们创建了一个README文件，并在里面写了一句话： 
this is readme。

下面我们修改这个文件： 
this is readme，modify。

接下来，我们使用git status命令来查看当前git仓库哪些内容被修改了：

```Bash
➜  gitTest git:(master) git status
# On branch master
# Changes not staged for commit:
#   (use "git add <file>..." to update what will be committed)
#   (use "git checkout -- <file>..." to discard changes in working directory)
#
#   modified:   README.txt
#
no changes added to commit (use "git add" and/or "git commit -a")123456789123456789
```

我们可以发现，git提示我们： 
modified: README.txt，更进一步，我们可以使用git diff命令来查看具体的修改：

```Bash
diff --git a/README.txt b/README.txt
index 2744f40..f312f1a 100644
--- a/README.txt
+++ b/README.txt
@@ -1 +1 @@
-this is readme
\ No newline at end of file
+this is readme, modify
\ No newline at end of file
(END)1234567891012345678910
```

这样就查看了具体的修改。

下面我们继续add、commit：

```Bash
➜  gitTest git:(master) ✗ git add README.txt
➜  gitTest git:(master) ✗ git status
# On branch master
# Changes to be committed:
#   (use "git reset HEAD <file>..." to unstage)
#
#   modified:   README.txt
#
➜  gitTest git:(master) ✗ git commit -m "modify readme"
[master 1629cdc] modify readme
 1 file changed, 1 insertion(+), 1 deletion(-)
➜  gitTest git:(master) git status
# On branch master
nothing to commit, working directory clean12345678910111213141234567891011121314
```

add和commit之后，我们都使用status来查看下状态，可以发现，在commit之后，git提示我们，工作区是干净的。

## 版本记录

在项目中，一个仓库通常可能有非常多次的add、commit过程，这些记录都会被记录下来，我们可以使用git log来查看这些记录：

```Bash
commit 1629cdcf2307bf26c0c5467e10035c2bd751e9d0
Author: xuyisheng <xuyisheng@hujiang.com>
Date:   Sun Jun 28 14:45:14 2015 +0800

    modify readme

commit c19081b6a48bcd6fb243560dafc7a35ae5e74765
Author: xuyisheng <xuyisheng@hujiang.com>
Date:   Sun Jun 28 14:35:00 2015 +0800

    add readme
(END)123456789101112123456789101112
```

每条记录都对应一个commit id，这是一个40个16进制的sha-1 hash code。用来唯一标识一个commit。

同时，我们也可以使用gitk命令来查看图形化的log记录：

![这里写图片描述](http://img.blog.csdn.net/20150628145302678)

git会自动将commit串成一个时间线。每个点，就代表一个commit。点击这些点，就可以看见相应的修改信息。

## 工作区与暂存区

Git通常是工作在三个区域上：

1. 工作区
2. 暂存区
3. 历史区

其中工作区就是我们平时工作、修改代码的区域，而历史区，用来保存各个版本，而暂存区，则是Git的核心所在。

暂存区保存在我们前面讲的那个.git的隐藏文件夹中，是一个叫index的文件。

![这里写图片描述](http://img.blog.csdn.net/20150628150331784)

当我们向Git仓库提交代码的时候。add操作实际上是将修改记录到暂存区，我们来看gitk：

![这里写图片描述](http://img.blog.csdn.net/20150628151835039)

可以发现，我们在本地已经生成了一个记录，但是还没有commit，所以当前HEAD并没有指向我们的修改，修改还保存在暂存区。

当我们commit之后，再看gitk：

![这里写图片描述](http://img.blog.csdn.net/20150628152116024)

这时候，HEAD就已经移到了我们的修改上，也就是说，我们的提交生成了一个新的commit。git commit操作就是将暂存区的内容全部提交。如果内容不add到暂存区，那么commit就不会提交修改内容。

> PS 这里需要说明一个概念，git管理的是修改，而不是文件，每个sha-1的值，也是根据内容计算出来的。

## 版本回退

如果这个世界一直只需要git add、commit，就不会有这么多的问题了，但是，愿望是美好的，现实是残酷的。如何处理版本的回退和修改，是使用git非常重要的一步。

### checkout && reset

我们来考虑几种情况：

1. 文件已经修改，但是还没有git add
2. 文件已经add到暂存区，又作了修改
3. 文件的修改已经add到了暂存区

分别执行以下操作：

```Bash
➜  gitTest git:(master) ✗ git checkout -- README.txt11
```

1. 修改被删除，完全还原到上次commit的状态，也就是服务器版本
2. 最后的修改被删除，还原到上次add的状态，也就是修改前的暂存区状态

总的来说，就是还原到上次add、commit的状态。

而对于第三种情况，我们可以使用git reset：

```Bash
➜  gitTest git:(master) ✗ git status
# On branch master
# Changes to be committed:
#   (use "git reset HEAD <file>..." to unstage)
#
#   modified:   README.txt
#
➜  gitTest git:(master) ✗ git reset HEAD README.txt
Unstaged changes after reset:
M   README.txt
➜  gitTest git:(master) ✗ git status
# On branch master
# Changes not staged for commit:
#   (use "git add <file>..." to update what will be committed)
#   (use "git checkout -- <file>..." to discard changes in working directory)
#
#   modified:   README.txt
#
no changes added to commit (use "git add" and/or "git commit -a")1234567891011121314151617181912345678910111213141516171819
```

通过git reset HEAD README.txt，我们就把暂存区的文件清除了，这样，在本地就是add前的状态，通过checkout操作，就可以进行修改回退了。

> PS : git checkout其实是用版本库里的版本替换工作区的版本，无论工作区是修改还是删除

### 回退版本

当我们的仓库有了大量的提交的时候，我们可以通过git log来查看（可以指定 –pretty=oneline 来优化显示效果）。

```Bash
e7ae095cafc8ddc5fda5a5d8b23d0bcaaf74ac39 modify again
5427c66703abfeaba3706f938317251ef2567e8b delete test.txt
08098a21a918cfbd6377fc7a03a08cac0e6bcef6 add new file
b687b06fbb66da68bf8e0616c8049f194f03a062 e
8038c502e6f5cbf34c8096eb27feec682b75410b update
34ad1c36b97f090fdf3191f51e149b404c86e72f modify again
1629cdcf2307bf26c0c5467e10035c2bd751e9d0 modify readme
c19081b6a48bcd6fb243560dafc7a35ae5e74765 add readme1234567812345678
```

那么我们如果回退到指定版本呢？在Git中，用HEAD表示当前版本，上一个版本就是HEAD^，上上一个版本就是HEAD^^，当然往上100个版本就不要这样写了，写成HEAD~100即可。

下面我们就可以回退了：

```Bash
➜  testGit git:(master) git reset --hard HEAD^
HEAD is now at 5427c66 delete test.txt1212
```

要回退到哪个版本，只要HEAD写对就OK了。你可以写commit id，也可以HEAD^，也可以HEAD^^。

### 前进版本

有时候，如果我们回退到了旧的版本，但是却后悔了，想回到后面某个新的版本，但这个时候，我的commit id已经忘了，怎么办呢？ 
没事，通过这个指令：

```Bash
5427c66 HEAD@{0}: reset: moving to HEAD^
e7ae095 HEAD@{1}: checkout: moving from dev to master
7986a59 HEAD@{2}: checkout: moving from master to dev
e7ae095 HEAD@{3}: checkout: moving from dev to master
7986a59 HEAD@{4}: checkout: moving from 7986a59bd8683acb560e56ff222324cd49edb5e5 to dev
7986a59 HEAD@{5}: checkout: moving from master to origin/dev
e7ae095 HEAD@{6}: clone: from git@github.com:xuyisheng/testGit.git12345671234567
```

这样我们就可以找到commit id用来还原了。

git reflog记录的就是你的操作历史。

## 文件操作

### git rm

如果我们要删除git仓库中的文件，我们要怎么做呢？

我们创建一个新的文件并提交，再删除这个文件：

```Bash
➜  gitTest git:(master) rm test.txt
➜  gitTest git:(master) ✗ git status
# On branch master
# Changes not staged for commit:
#   (use "git add/rm <file>..." to update what will be committed)
#   (use "git checkout -- <file>..." to discard changes in working directory)
#
#   deleted:    test.txt
#
no changes added to commit (use "git add" and/or "git commit -a")1234567891012345678910
```

git提示我们delete一个文件，这时候你可以选择:

```Bash
➜  gitTest git:(master) ✗ git checkout test.txt11
```

这样就撤销了删除操作，或者使用：

```Bash
➜  gitTest git:(master) ✗ git rm test.txt
rm 'test.txt'
➜  gitTest git:(master) ✗ git status
# On branch master
# Changes to be committed:
#   (use "git reset HEAD <file>..." to unstage)
#
#   deleted:    test.txt
#
➜  gitTest git:(master) ✗ git commit -m "delete test.txt"
[master 5427c66] delete test.txt
 1 file changed, 0 insertions(+), 0 deletions(-)
 delete mode 100644 test.txt1234567891011121312345678910111213
```

通过git rm指令，删除文件，最后提交修改，就真实的删除了文件。

### 文件暂存

这里的暂存不是前面说的暂存区，而是只一次备份与恢复操作。举个例子，当前我们在dev分支上进行一个新功能的开发，但是开发到一半，[测试](http://lib.csdn.net/base/softwaretest)提了一个issue，这时候，我们需要创建一个issue分支来修改这个bug，但是，当前dev分支是不干净的，新功能开发到一半，直接从dev上拉分支，代码是不完善的，可能会编译不过。 
so，你可以使用：

```Bash
git stash11
```

指令来将当前修改暂存，这样就可以切换到其他分支或者就在当前干净的分支上checkout了。 
比如你checkout了一个issue分支，修改了bug，使用git merge合并到了master分支，删除issue分支，切换到dev分支，想继续之前的新功能开发。 
这时候，就需要恢复现场了：

首先，通过：

```Bash
git stash list11
```

我们可以查看当前暂存的内容记录。

然后，通过git stash apply或者git stash pop来进行恢复，它们的区别是，前者不会删除记录（当然你可以使用git stash drop来删除），而后者会。

## 远程仓库

既然git是分布式的仓库管理，那么我们肯定是需要多台服务器进行各种操作的，一般在开发中，我们会用一台电脑做中央服务器，各个终端从中央服务器拉取代码，提交修改。

那么我们如何去搭建一个git远程服务器呢，答案是不要搭建，个人开发者可以通过github来获取免费的远程git服务器，或者是国内的开源中国之类的，同样也提供免费的git服务器，而对于企业用户，可以通过gitlab来获取git远程服务器。

### 身份认证

当本地git仓库与git远程仓库通信的时候，需要进行SSH身份认证。

打开根目录下的.ssh目录：

```Bash
➜  ~  cd .ssh
➜  .ssh  ll
total 40
-rw-------  1 xys  staff   1.7K  5 15 23:53 github_rsa
-rw-r--r--  1 xys  staff   402B  5 15 23:53 github_rsa.pub
-rw-------  1 xys  staff   1.6K  5 15 09:38 id_rsa
-rw-r--r--  1 xys  staff   409B  5 15 09:42 id_rsa.pub
-rw-r--r--  1 xys  staff   2.0K  6  3 13:34 known_hosts1234567812345678
```

如果没有id_rsa和id_rsa.pub这两个文件，就通过如下的命令生成：

```Bash
ssh-keygen -t rsa -C "youremail@example.com"11
```

id_rsa和id_rsa.pub这两个文件，就是SSH Key的秘钥对，id_rsa是私钥，不能泄露出去，id_rsa.pub是公钥，用在github上表明身份。

在github的ssh key中增加刚刚生成的key：

![这里写图片描述](http://img.blog.csdn.net/20150628161951800)

### 同步协作

现在你在本地建立了git仓库，想与远程git仓库同步，这样github的远程仓库可以作为你本地的备份，也可以让其他人进行协同工作。

我们先在github上创建一个repo（仓库）：

![这里写图片描述](http://img.blog.csdn.net/20150628162418735)

创建之后，github给我们提示：

![这里写图片描述](http://img.blog.csdn.net/20150628162552260)

github告诉了我们如何在本地创建一个新的repo或者将既存的repo提交到远程git。

由于我们已经有了本地的git，所以，安装提示：

```
➜  gitTest git:(master) git remote add origin git@github.com:xuyisheng/testGit.git
➜  gitTest git:(master) git push -u origin master
Counting objects: 18, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (8/8), done.
Writing objects: 100% (18/18), 1.39 KiB | 0 bytes/s, done.
Total 18 (delta 1), reused 0 (delta 0)
To git@github.com:xuyisheng/testGit.git
 * [new branch]      master -> master
Branch master set up to track remote branch master from origin.1234567891012345678910
```

现在我们再看github上的仓库：

![这里写图片描述](http://img.blog.csdn.net/20150628162936736)

README已经提交上去了。

git remote add origin git@github.com:xuyisheng/testGit.git这条指令中的origin，就是远程仓库的名字，你也可以叫别的，但是默认远程仓库都叫做origin，便于区分。

> PS: 这里还需要注意下的是git push的-u参数，加上了-u参数，Git不但会把本地的master分支内容推送的远程新的master分支，还会把本地的master分支和远程的master分支关联起来。不过后面的push就不需要这个参数了。

之后我们再做修改：

```Bash
➜  gitTest git:(master) ✗ git add README.txt
➜  gitTest git:(master) ✗ git commit -m "modify again"
[master e7ae095] modify again
 1 file changed, 2 insertions(+), 1 deletion(-)
➜  gitTest git:(master) git push
Counting objects: 5, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 285 bytes | 0 bytes/s, done.
Total 3 (delta 0), reused 0 (delta 0)
To git@github.com:xuyisheng/testGit.git
   5427c66..e7ae095  master -> master123456789101112123456789101112
```

可以直接使用git push或者git push origin master来指定仓库和分支名。

### clone远程仓库

记得我们之前说的，创建本地仓库的几种方式，其中有一种是clone：

我们可以通过git clone指令来clone一个远程仓库：

![这里写图片描述](http://img.blog.csdn.net/20150628164058747)

下面就显示了远程仓库的地址，一般我们使用SSH的方式，当然，也可以使用其他方式，然并卵。

> PS: 使用https除了速度慢以外，还有个最大的麻烦是每次推送都必须输入口令，但是在某些只开放http端口的公司内部就无法使用ssh协议而只能用https。

直接使用：

```
➜  MyWork  git clone git@github.com:xuyisheng/testGit.git
Cloning into 'testGit'...
remote: Counting objects: 21, done.
remote: Compressing objects: 100% (9/9), done.
remote: Total 21 (delta 1), reused 21 (delta 1), pack-reused 0
Receiving objects: 100% (21/21), done.
Resolving deltas: 100% (1/1), done.
Checking connectivity... done1234567812345678
```

## 分支管理

个人认为，创建分支是git最大的魅力。 
git中的分支就好像现在的平行宇宙，不同的分支互不干扰，相互独立，你就像一个上帝一样，可以随时对任意一个分支进行操作，可以今天去这个branch玩，明天去另一个branch，玩腻了，再把两个分支合并，一起玩。

举个比较恰当的例子，我现在要开发一个新功能，需要3个月的时间，但是我不能每天都把未完成的代码提交到大家都在用的分支上，这样人家拉取了我的代码就无法正常工作了，但是我又不能新建一个仓库，这也太浪费了，所以我可以新建一个分支，在这个分支上开发我的功能，同时能够备份我的代码，当开发完毕后，直接合并分支，整个新功能就一下子加入了大家的分支。

### 创建分支

你的每次提交，Git都把它们串成一条时间线，这条时间线就是一个分支。不创建分支时，只有一条时间线，在Git里，这个分支叫主分支，即master分支。HEAD严格来说不是指向提交，而是指向master，master才是指向提交的，所以，HEAD指向的就是当前分支。

![这里写图片描述](http://img.blog.csdn.net/20150628165316836)

这里的时间线就看的非常清楚了。

我们通过如下指令来创建分支：

```Bash
➜  gitTest git:(master) git checkout -b dev
Switched to a new branch 'dev'
➜  gitTest git:(dev)123123
```

-b参数代表创建并切换到该分支，相当于：

```Bash
$ git branch dev
$ git checkout dev
Switched to branch 'dev'123123
```

不加-b参数就是直接切换到已知分支了。

### 查看分支

通过git branch指令可以列出当前所有分支：

```Bash
➜  gitTest git:(dev) git branch
* dev
  master123123
```

当前分支上会多一个*

### 合并分支

切换到dev分支后，我们进行修改，add并commit：

此时我们再切换到master分支，再查看当前修改，你会发现，dev分支上做的修改这里都没有生效。必须的，不然平行宇宙就不平行了。我们通过如下指令来进行分支的合并：

```Bash
➜  gitTest git:(master) git merge dev
Updating e7ae095..7986a59
Fast-forward
 README.txt | 3 ++-
 1 file changed, 2 insertions(+), 1 deletion(-)1234512345
```

这样再查看master分支下的文件，dev上的修改就有了。

### 删除分支

使用完毕后，我们不再需要这个分支了，所以，放心的删除吧：

```Bash
➜  gitTest git:(master) git branch -d dev
Deleted branch dev (was 7986a59).
➜  gitTest git:(master) git branch
* master12341234
```

> PS : 当分支还没有被合并的时候，如果删除分支，git会提示： 
> error: The branch ‘feature-vulcan’ is not fully merged. 
> If you are sure you want to delete it, run ‘git branch -D dev. 
> 也就是提示我们使用-D参数来进行强行删除。

看完分支的操作，有人可能会问，创建这么多分支，git会不会很累，当然不会，git并不是创建整个文件的备份到各个分支，而是创建一个指针指向不同的分支而已。切换分支，创建分支，都只是改变指针指向的位置。

分支是非常好的团体协作方式，一个项目中通常会有一个master分支来进行发布管理，一个dev分支来进行开发，而不同的开发者checkout出dev分支来进行开发，merge自己的分支到dev，当有issue或者新需求的时候，checkout分支进行修改，可以保证主分支的安全，即使修改取消，也不会影响主分支。

### 查看远程分支

当你从远程仓库克隆时，实际上Git自动把本地的master分支和远程的master分支对应起来了，并且，远程仓库的默认名称是origin。 
通过如下指令，我们可以查看远程分支：

```Bash
➜  gitTest git:(master) git remote
origin1212
```

或者：

```Bash
➜  gitTest git:(master) git remote -v
origin  git@github.com:xuyisheng/testGit.git (fetch)
origin  git@github.com:xuyisheng/testGit.git (push)123123
```

来显示更详细的信息。

### 推送分支

要把本地创建的分支同步到远程仓库上，我们可以使用：

```Bash
➜  gitTest git:(master) git checkout -b dev
Switched to a new branch 'dev'
➜  gitTest git:(dev) git push origin dev
Everything up-to-date12341234
```

这样就把一个dev分支推送到了远程仓库origin中。

### 抓取远程分支

当我们将远程仓库clone到本地后即使远程仓库有多个分支，但实际上，本地只有一个分支——master。

```Bash
➜  MyWork  git clone git@github.com:xuyisheng/testGit.git
Cloning into 'testGit'...
remote: Counting objects: 24, done.
remote: Compressing objects: 100% (11/11), done.
remote: Total 24 (delta 2), reused 23 (delta 1), pack-reused 0
Receiving objects: 100% (24/24), done.
Resolving deltas: 100% (2/2), done.
Checking connectivity... done
➜  MyWork  cd testGit
➜  testGit git:(master) git branch
* master12345678910111234567891011
```

现在，我要在dev分支上开发，就必须创建远程origin的dev分支到本地，用这个命令创建本地dev分支：

```Bash
➜  testGit git:(master) git checkout -b dev origin/dev
Branch dev set up to track remote branch dev from origin.
Switched to a new branch 'dev'
➜  testGit git:(dev)12341234
```

这样就把本地创建的dev分支与远程的dev分支关联了。 
后面你可以使用git push origin dev继续提交代码到远程的dev分支。

这种情况下，如果其他人也提交了到dev分支，那么你的提交就会与他的提交冲突，因此，你需要使用git pull先将远程修改拉取下来，git会自动帮你在本地进行merge，如果没有冲突，这时候再提交，就OK了，如果有冲突，那么必须手动解除冲突再提交。

## Tag

Tag的概念非常类似于branch。但是branch是可以不断改变、merge的，Tag不行，Tag可以认为是一个快照，用于记录某个commit点的历史快照。

### 创建标签

非常简单：

```Bash
➜  testGit git:(master) git tag version111
```

默认Tag会打在最后的提交上。但是你也可以通过commit id来指定要打的地方。

```Bash
➜  testGit git:(master) git tag version0 b687b06fbb66da68bf8e0616c8049f194f03a062
➜  testGit git:(master) git tag
version0
version112341234
```

> PS : 实际上commit id不需要写很长，通过前6、7位，git就可以查找到相应的id了。

还可以创建带有说明的标签，用-a指定标签名，-m指定说明文字：

```Bash
git tag -a v1 -m "version1" b687b06fbb66da68bf8e0616c8049f194f03a06211
```

通过如下指令来查看详细信息：

```Bash
git show <tagname>11
```

### 查看标签

```Bash
➜  testGit git:(master) git tag
version11212
```

### 删除标签

-d参数就可以了，与branch类似。

```Bash
➜  testGit git:(master) git tag
version0
version1
➜  testGit git:(master) git tag -d version0
Deleted tag 'version0' (was b687b06)
➜  testGit git:(master) git tag
version112345671234567
```

### 推送到远程

将本地tag推送到远程参考上：

```Bash
➜  testGit git:(master) git push origin version0
Total 0 (delta 0), reused 0 (delta 0)
To git@github.com:xuyisheng/testGit.git
 * [new tag]         version0 -> version012341234
```

或者：

```Bash
➜  testGit git:(master) git push origin --tags
Total 0 (delta 0), reused 0 (delta 0)
To git@github.com:xuyisheng/testGit.git
 * [new tag]         version1 -> version112341234
```

来push所有的tag。

### 删除远程Tag

当Tag已经push到远程仓库后，我们要删除这个Tag，首先需要先删除本地Tag：

```Bash
➜  testGit git:(master) git tag -d version0
Deleted tag 'version0' (was b687b06)1212
```

再push到远程，带指令有所不同：

```Bash
➜  testGit git:(master) git push origin :refs/tags/version0
To git@github.com:xuyisheng/testGit.git
 - [deleted]         version0123123
```
