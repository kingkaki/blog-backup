---
title: Git 安装与操作
url: 164.html
id: 164
categories:
  - 学习笔记
  - 搭建与配置
date: 2018-01-19 17:24:40
tags:
---

不得不说，在大一开始就听说有github那么一个牛逼的社区平台，后来也在上面借鉴过许多大佬们的源码，也十分感谢各位优秀的行业人员的无私奉献。不过，至今为止，个人还是不会使用git命令，就连之前上传代码什么的也仅仅只是通过网页点击的方式，没有感到一丝便利的地方。因此，决定好好学习一下git的操作。  

### Git 安装

其实不能再简单了 **centos下**

yum install git

**windows下** 去官网下载git的程序安装即可  

### Git 设置姓名、邮箱

对于git 来说，每台机器都要自保家门，说出你的名字和邮箱，否则在后面难以继续操作

 git config --global user.name "Your Name"
 git config --global user.email "email@example.com"

注意`git config`命令的`--global`参数，用了这个参数，表示你这台机器上所有的Git仓库都会使用这个配置，当然也可以对某个仓库指定不同的用户名和Email地址。  

### 工作区与暂存区

无耻的拿了一下廖大神的网站上来，不得不说，讲的很详细，就不再写一遍了 [https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/0013745374151782eb658c5a5ca454eaa451661275886c6000](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/0013745374151782eb658c5a5ca454eaa451661275886c6000)  

### 基础常用操作

#### 新建版本库

1、在本地新建一个文件夹

mkdir test

2、切换到对应目录，并将该目录变成git可以管理的仓库

cd test
git init

 

#### 添加文件到仓库

例如我们在版本库的文件夹中创建了 read.md test.txt 两个文件，现在我们想把他们确认添加到git仓库中 1、add 文件修改添加到暂存区

git add read.md test.txt

git add -A

git add 文件名   也可以通过 git add -A 或者 git add --all 添加全部更新 2、commit 把暂存区的所有内容提交到当前分支

git commit -m "writea a readme and test file"

git commit -m 备注  

#### 查看仓库当前状态

git status

该命令告诉我们从上一次commit之后到现在，仓库的更改状况（是否被修改等）  

#### 查看具体的修改内容

git diff

git diff readme.md

可以制定查看某个文件的修改内容，也可以查看全部的修改内容  

#### 查看历史版本

git log

git log --pretty=oneline

会显示各个版本的id号，以及备注 用第二种方式可以单行输出，稍微直观一些  

#### 回退之前的版本

首先，Git必须知道当前版本是哪个版本，在Git中，用HEAD表示当前版本，也就是最新的提交，上一个版本就是HEAD^，上上一个版本就是HEAD^^，当然往上100个版本写100个^比较容易数不过来，所以写成HEAD~100
git reset --hard HEAD^

这样之后，就可以回退到之前的版本  

#### 取消回退，还原至未更新前版本

取消回退可能没有回退那么简单，它有一个重要的参数就是commit id，就是我们之前git log看到的id，但是回退后将无法查看之前的id 1、查看之前版本commit id

\[root@localhost test\]# git reflog
0449a59 HEAD@{0}: HEAD^: updating HEAD
96f69f4 HEAD@{1}: commit: second commit

会显示所有的commit id，找到你所对应的id号 2、reset回退

git reset --hard 96f69f4

后面的参数即为之前的commit id 这样就可以取消之前的回退，退至指定位置  

#### 撤销修改

###### 丢弃工作区的内容（未add）

git checkout -- read.md

git checkout -- 文件名

###### 丢弃暂存区的内容(add但未commit)

git reset HEAD read.md
git checkout -- read.md

git reset HEAD 文件名 git checkout -- 文件名  

#### 删除文件

当我们在本地通过rm -rf test.txt删除了文件之后需要同步到版本库中，这里就会与之前有些不同 1、git rm

git rm test.txt

2、提交

git commit -m "drop test.txt"

假如是误删，则可以通过checkout 的方式恢复

checkout -- test.txt

     

### Git 绑定ssh key

绑定ssh key 的目的是为了让本地主机授权执行远程代码库的git操作命令。因此在操作之前，应设置 ssh key 1、在centos中，切换到~/.ssh 目录中，生成ssh公钥和私钥

cd  ~/.ssh/
ssh-keygen

2、然后进入，github的设置页面，新建一个ssh key ![](http://blog.kingkk.com/wp-content/uploads/2018/01/74ed16a9a3ad011022cb8a1bfe20441c.png) 3、将之前生成的公钥内容放入其中 ![](http://blog.kingkk.com/wp-content/uploads/2018/01/8757d82c8747c22cd2b4562d86136092.png)  

### 关联本地库

1、先在github中任意新建一个测试版本库，并找到对应的git地址 ![](http://blog.kingkk.com/wp-content/uploads/2018/01/3ee5bead3b40dc51f43a30edf2837a09.png) 2、关联绑定

git remote add origin git@github.com:kingkaki/test.git

这样就可以将远程的库与本地的库相互关联绑定 3、推送本地内容

git push -u origin master

-u 参数仅需在第一次推送时添加即可    

#### 克隆仓库

1、依旧是要获取之前的ssh地址  git@github.com:kingkaki/py_script.git 2、git clone

git clone git@github.com:kingkaki/py_script.git

即可以将远程库克隆到本地进行操作    

### Git 分支（branch）

 

##### 查看分支：`git branch`

\[root@localhost test\]# git branch
\* master

git的主分支为master    *表示当前分支

##### 创建分支：`git branch <name>`

\[root@localhost test\]# git branch dev 
\[root@localhost test\]# git branch
 dev
\* master

可以看到新后多了一个 dev分支 创建+切换分支：`git checkout -b <name>`  

##### 切换分支：`git checkout <name>`

git checkout dev

切换到dev分支

##### 合并某分支到当前分支：`git merge <name>`

git merge master

将master分支与当前分支合并  

##### 删除分支：`git branch -d <name>`

\[root@localhost test\]# git branch -d master
Deleted branch master (was 5fa0562).

成功删除dev分支  

##### 保存分支信息合并

git merge --no-ff -m "merge with no-ff" dev

在合并的同时，加上**--no-ff** 参数    

#### 暂存工作区

当当前工作区操作至一半时，想去执行别的操作，就需要暂时将工作区的工作暂存着

git stash

##### 查看暂存的工作区

\[root@localhost test\]# git stash list
stash@{0}: WIP on dev: 228c5e1 test brach

就可以看到当前暂存的工作区

##### 还原暂存的工作区

`**git stash apply**`恢复，但是恢复后，stash内容并不删除，需要用`git stash drop`来删除 **`git stash pop`**恢复后 stash内容删除      

### Git 标签

##### 新建标签

命令`git tag <name>`用于新建一个标签，默认为`HEAD`，也可以指定一个commit id

git tag v1.0
git tag v0.9 0449a59

为当前版本打上v1.0标签，为id为0449a59的打上v0.9标签

##### 查看所有标签`git tag`

\[root@localhost test\]# git tag
list
v0.9
v1.0

##### 删除本地标签 `git tag -d <tagname>`

\[root@localhost test\]# git tag -d list
Deleted tag 'list' (was 228c5e1)

##### 推送本地标签 `git push origin <tagname>`

##### 推送全部未推送过的标签 `git push origin --tags`

##### 删除远程标签 `git push origin :refs/tags/<tagname>`

   

### .gitignore文件

当你不想把本地所有的文件都提交到库时，比如一些配置密码文件之类，这时候可以建立一个.gitignore文件，忽略掉那些你不想上传的文件 网上的忽略文件原则

1.  忽略操作系统自动生成的文件，比如缩略图等；
2.  忽略编译生成的中间文件、可执行文件等，也就是如果一个文件是通过另一个文件自动生成的，那自动生成的文件就没必要放进版本库，比如Java编译产生的`.class`文件；
3.  忽略你自己的带有敏感信息的配置文件，比如存放口令的配置文件。

还有网上大神写好的模板。真是太良心了。。[https://github.com/github/gitignore](https://github.com/github/gitignore) 还有廖大神的一个简单的python版本,可以参考一下

\# Windows:
Thumbs.db
ehthumbs.db
Desktop.ini

\# Python:
*.py\[cod\]
*.so
*.egg
*.egg-info
dist
build

\# My configurations:
db.ini
deploy\_key\_rsa

最后记得将该.gitignore文件也提交到git

##### 强制添加至git(无视.gitignore文件）`git add -f <filename>`

##### 检查.gitignore文件 `git check-ignore`

发现.gitignore写错时，可以用该命令来检查

$ git check-ignore -v App.class
.gitignore:3:*.class App.class

 

* * *

最后，就是多实践吧，先多操作，用熟练了一个就好了 主要命令应该就

*   git add filename
*   git commit -m "xxxx"
*   git clone xxxxxxxx
*   git push origin master

或者再加两个

*   git diff
*   git status

最后，多使用，并在此感谢廖雪峰的教程