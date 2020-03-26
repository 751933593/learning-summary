### GitHub

#### 1.版本控制

#### 2.Git简介

#### 3.Git命令行操作

##### 3.1 本地库初始化

```shell
# 生成.git目录，管理整个项目
git init
```

##### 3.2 设置签名

作用：用于区分不同的开发人员

用户级别：项目级别和系统用户级别

```powershell
# 项目级别
git config user.name capture
git config user.email 12345@qq.com
# 系统用户级别
git config --global user.name global_capture
git config --global user.email global_12345@qq.com

cat .git/cinfg
cat ~/gitconfig
```

##### 3.3 将文件添加到暂存区

```powershell
# 查看文件状态
git status
# 将文件添加到暂存区
git add aa.txt
# 将文件从暂存区移除
git rm --cached aa.txt
```

##### 3.4 提交暂存区中的文件

```powershell
# 提交暂存区中的文件
git commit -m "my first commit" aa.txt
```

##### 3.5 查看历史版本

```shell
# 以每个版本一行的形式显示
git log --pretty=oneline
git log --oneline
git reflog
```

##### 3.6 版本前进与后退

```shell
# 前进或后退到指定版本
git reset --hard [版本号]
```

reset 的三个参数对比：

- --soft 在本地库移动HEAD指针
- --mixed 在本地库和暂存区移动HEAD指针
- --hard 在本地库、暂存区和工作区移动HEAD指针

##### 3.7 对比文件差异

```powershell
# 对比工作区中所有文件
git diff
# 工作区 vs 暂存区
git diff file
# 本地库(可以是历史版本) vs 工作区
git diff HEAD [版本号] file1
```



##### 3.8 分支管理

```shell
# 查看分支
git branch -v
# 创建分支
git branch branch_01
# 删除本地分支
git branch -D branch_01
# 切换分支
git checkout branch_01
# 合并分支
# 1.切换到master分支 git checkout master
# 2.将master与branch_01合并 git merge branch_01
# 冲突解决
# 1.查看特殊符号并手动解决冲突
# 2.解决后 git add file  和 git commit -m "resolve confilict"
```



#### 4.Git的基本原理

- 集中式版本控制是保存每个版本比上个版本的变化量
- Git是每个版本一个快照，如果文件未修改，则该版本的快照指向上一个版本

##### 4.1 版本的内部构造

![](..\img\GitHub\Git版本的内部构造.jpg)

##### 4.2 分支管理机制

![](..\img\GitHub\Git分支管理机制.jpg)

- 分支和HEAD是指针

#### 5.Git图形化界面操作

##### 5.1 本地库推送到远程库

```powershell
# 保存远程库地址并创建别名（https的方式）
git remote add 远程库别名 远程库地址
# 查看远程库信息
git remote -v
# 将本地库推到远程库的某个分支
git push 远程库别名 master
```

##### 5.2 从远程库克隆到本地库

```powershell
# 从远程库克隆到本地库
git clone 远程库某个仓库的克隆地址
```

##### 5.3 团队合作操作仓库

1. 首先仓库创建者（管理者向团队内其他成员发送邀请）
2. 其他成员接受邀请
3. 其他成员可以向远程库提交版本 git push 远程库别名 远程库分支名
4. 拉取远程库的信息
   - pull = fetch + merge （git pull 远程库别名 远程库分支名）
   - 将远程库拉取下来：   git fetch 远程库别名 远程库分支名
   - 查看远程库的信息：   1.切换到远程库的具体分支   git checkout 远程库别名:远程库分支名  2.查看文件
   - 将远程库拉取下来的信息与本地库合并：    git merge 远程库别名/远程库分支名
5. 拉取后出现冲突
   0. 如果出现冲突则不能提交
   1. 然后拉取后会进入冲突模式
   2. 解决冲突后提交到本地库，push到远程库

##### 5.4 跨团队协作

![](..\img\GitHub\Git项目跨团队协作开发.jpg)

##### 5.5 SSH免密登录

```powershell
# 1.进入用户家目录
cd ~
# 2.删除.ssh目录
rm -rf .ssh
# 3.生成ssh秘钥目录
ssh-keygen -t rsa -C 邮箱名
# 4.进入.ssh目录查看文件，复制文件id_rsa.pub中的内容
cd .ssh
cat id_rsa.pub
# 5.登录GitHub，进入settings->SSH and GPG keys
# 6.New SSH Key
# 7.输入复制的秘钥信息
# 8.回到Git bash创建远程库地址别名
git remote add 远程库别名 远程库ssh地址
# 9.推送文件进行测试
git push 远程库别名 远程库分支名
```

#### 6.Git工作流

##### 6.1 集中式工作流

大家都向远程仓库的master分支执行push/pull操作，唯一和SVN不同的是每个人都有本地仓库

![](..\img\GitHub\集中式工作流.jpg)

##### 6.2 GitFlow工作流

![](..\img\GitHub\GitFlow工作流.jpg)

##### 6.3 Forking工作流



#### 7.GitLab服务器环境搭建

- 操作系统：centos7

- 官网：https://about.gitlab.com
- centos7安装地址：https://about.gitlab.com/install/#centos-7



