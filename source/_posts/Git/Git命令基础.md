---
title: Git命令基础
#tags: [hexo建站,hexo部署,github部署,个人博客]      #添加的标签
categories: 
  - Git
description: git一些基础的命令
#cover: 
---

## 创建版本库
首先创建一个文件夹，然后cd到该文件夹下，输入以下命令：
```text
git init
```
当前目录下会多了一个.git的目录，让git帮助我们管理当前文件夹。



## 版本穿梭

远程仓库的默认名称是`origin`
远程仓库主分支`origin master`
本地仓库主分支`master`

### git区域

- 工作区：就是你在电脑里能看到的目录
- 暂存区：一般存放在 .git 目录下的 index 文件（.git/index）中，所以我们有时也把暂存区叫作索引（index）
- 版本库：工作区有一个隐藏目录.git

管理文件（红变绿）：

```text
git add 文件名	（*代表全部文件）
```

提交一条信息命令，**生成一个版本**：

```text
git commit -m ""
```

git add把文件添加进去，实际上就是把**工作区**文件修改添加到**暂存区**；
git commit是一次性提交暂存区的全部更改，实际上就是把**暂存区**的所有内容提交到**本地的当前分支（master）**。

![git img4](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/git%20img4.png)



查看版本记录：

```text
git log
```






### 版本回退与撤销修改

穿梭前，用`git log`可以查看提交历史，以便确定要回退到哪个版本。（只能看到当前版本之前的历史版本）
要重返未来，用`git reflog`查看历史命令，以便确定要回到未来的哪个版本。（可以看到全部版本）

#### 工作区

当改乱了工作区某个文件的内容，想直接丢弃工作区的修改时，用命令`git checkout -- file`。

#### 暂存区

当工作区某个文件的内容添加到了暂存区时（add），想撤销暂存，用命令`git reset HEAD <file>`或者`git restore --staged 文件名`。

#### 版本库

已经提交了不合适的修改到版本库时，想要撤销本次提交，并且回到之前任一个版本，使用命令`git reset --hard commit_id`，这时，如果想推送到远程库，必须使用`git push -f `强制推送。

reset参数：

- –soft：只更新版本库内容
- –mixed：更新版本库、暂存区内容
- –hard：更新版本库、暂存区内容

<u>注：`git checkout -- file`也可以用来恢复工作区删除了的文件，前提是该文件已经提交到版本库。</u>



### 删除文件

方法一：将已提交到暂存区的某个文件从工作区中删除，执行`rm -rf a.txt  # 删除文件`，再执行`git add .`完成删除。

方法二：使用 `git rm a.txt` 命令删除已跟踪的文件。



## 分支

![git分支](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/git%E5%88%86%E6%94%AF.png)

### 创建分支

```text
创建分支: git branch 分支名
切换分支: git checkout 分支名  或者 git switch 分支名
查看所有分支: git branch -a
分支合并: git merge 要合并的分支名 (把分支合并到当前分支)
（合并分支有冲突，则手动把冲突解决，再提交）
```

注：创建分支后，由于新建立的分支还未在git上，所以push的时候，需要用命令：git push --set-upstream origin hexo。



### 删除分支

```text
首先使用git branch -a查看当前所有分支。
删除本地分支：git branch -d 分支1[,分支名2,分支名3...]
删除远程分支: git push origin --delete 分支名1[,分支命2,分支名...]
```

注：删除分支时，当前分支不能停留在要删除的分支上，要切换到其他任意分支，再去删除目标分支



### 缓存分支修改

​		有时候，我们可能会同时在几个分支上进行开发，在切换分支的时，如果当前分支有修改，这个时候功能还没有全部开发完成，你并不想马上提交到版本库的话，我们可以使用Git提供的缓存修改的命令，把这部分修改暂时缓存起来，切换回这个分支的时候，再把它取出来。

```text
缓存修改命令：git stash 
```
【注：这个命令会把提交到暂存区，就是把使用git add提交之后的代码缓存起来，对未经Git管控的文件修改并不做缓存】
```text
查看缓存列表:git stash list
从缓存中恢复修改并删除缓存内容: git stash pop
```

### 同步主分支上的修改
在master分支上修复的bug，想要合并到当前dev分支，可以用`git cherry-pick <commitId>`命令，把bug提交的修改“复制”到当前分支，避免重复劳动。



## 解决冲突

主分支master和feature1分支都同时修改了同一个文件readme.txt，如下图所示：
![git img5](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/git%20img5.png)

当合并两个分支的时，这个文件就会存在冲突，必须手动解决
```text
$ git merge feature1
Auto-merging readme.txt
CONFLICT (content): Merge conflict in readme.txt
Automatic merge failed; fix conflicts and then commit the result.
```
可以直接查看readme.txt的内容：
```text
Git is a distributed version control system.
Git is free software distributed under the GPL.
Git has a mutable index called stage.
Git tracks changes of files.
<<<<<<< HEAD
Creating a new branch is quick & simple.
=======
Creating a new branch is quick AND simple.
>>>>>>> feature1
```
Git用<<<<<<<，===，>>>>>>>标记出不同分支的内容。




## 拉取

&emsp; git分为本地仓库和远程仓库，我们一般情况都是写完代码，commit到本地仓库（生成本地仓的commit ID，代表当前提交代码的版本号），然后push到远程仓库（记录这个版本号）。
&emsp; 假设我们本地仓库的 master 分支上 commit ID =1 ，orign/master中的commit ID =1 ;这时候远程仓库别人更新了github ogirn库中master分支上的代码，新的代码版本号commit ID =2 ,那么在github上 orign/master的commitID=2，然后我们要更新代码。git流程示意图如下：
![git img1](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/git%20img1.png)

### git fetch
- 使用`git fetch`更新代码，本地的库中master的commitID不变，还是等于1。但是与git上面关联的那个orign/master的commit ID变成了2。这时候本地相当于存储了两个代码的版本号，还要通过`git merge orign/master`去合并这两个不同的代码版本，如果这两个版本都修改了同一处的代码，这时候merge就会出现冲突，然后我们解决冲突之后就生成了一个新的代码版本。
- 这时候本地的代码版本可能就变成了commit ID=3，即生成了一个新的代码版本。
![git img2](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/git%20img2.png)

### git pull
- 使用`git pull`会将最新代码更新到本地，head和remotes中的commitID同时被更新为远程分支最新的commitID。
![git img3](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/git%20img3.png)
