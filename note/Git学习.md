[TOC]

# Git学习

## 1、Git简介

Git是目前世界上最先进的分布式版本控制

工作区，暂存区，版本区简易示意图

```sequence
工作区->暂存区: git add
暂存区->版本区: git commit
版本区->暂存区: git reset HEAD
版本区->服务器: git push
服务器-->工作区: git pull
暂存区->工作区: git checkout -- file 
版本区-->工作区: git checkout HEAD file

Note right of 服务器:Gitee、Github等
Note right of 版本区:已经被Git版本控制
Note right of 暂存区:暂时保存，将交给Git进行版本管理
Note right of 工作区:项目文件夹，进行操作的区域
```



## 2、Git使用

确认用户名与邮箱

```bash
git config --global user.name username
git config --global user.email email
```

初始化版本库

```bash
git init
```

目录上多出一个.git的文件夹,这个文件夹是Git来跟踪管理版本库的

添加指定文件到暂存区中

```bash
git add xxx
```

 提交暂存区所有文件到版本区

```bash
git commit -m 'xxx'
```

版本回退与撤销

```bash
git reset --hard HEAD^
#三区文件全部回退一个版本

git reset --hard asadfa
#三区文件回退到asadfa的commit id版本

git reset HEAD
#用版本库中的文件去替换暂存区的全部文件

git checkout -- file
#暂存区指定文件去替换工作区的指定文件

git checkout HEAD file
#用版本库中的文件替换暂存区和工作区的文件
```

分支操作

```bash
git checkout -b dev
#创建dev分支，并切换到dev分支

git branch
#查看当前分支

git checkout master
#切换到master分支

git merge dev
#合并dev分支到当前分支

git checkout -d dev
#删除指定分支
```

版本冲突

合并分支时，如果在同一个文件，在同一个地方，都修改了或新增内容会引起版本冲突。

借助ide实现简单高效处理

## 3、GitHub使用

借助GitHub实现分享代码和配合开发

GitHub远程仓库的使用

本地有仓库，要和远程仓库做关联

```bash
git init 
git add .
git commit -m 'version: 1'
在GitHub上创建远程仓库
git remote add origin https
```

完成关联后，本地有仓库有内容，要推送给远程库

```bash
git push -u master
#第一次推送需要交 -u
```

将远程仓库内容拉取到本地

```bash
git pull origin master
#将远程仓库的master分支上代码版本复制/合并到本地master分支

git fetch origin master:tmp
#新建一个tmp分支，将远程仓库的master分支上代码版本复制到tmp分支上，不会自动合并
```

克隆仓库

```bash
git clone https
```

查看远程仓库信息

```bash
git remote -v
```

fork可以实现将别人的项目clone一份，但是owner变成自己

GitHub Pages的使用

1.推送本地代码到GitHub仓库

目录第一层需要存在index.html文件作为启动文件

2.配置GitHub Pages选项

settings中将GitHub Pages选项中的Source改为master branch

3.编辑主页的项目描述与演示网址

