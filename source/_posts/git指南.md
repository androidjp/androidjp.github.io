---
title: git指南
date: 2019-09-27 14:42:07
categories:
- 版本管理
tags:
- git
---

![](/images/git_01.png)

<!--more-->

# 克隆
* 克隆到githome目录
    ```
    git clone ssh://xxx.com:myProject ./githome
    ```
* 只克隆Dev分支到本地
    ```
    git clone -b Dev ssh://xxx.com:myProject
    ```

# 拉取与推送
## fetch 和 pull 的区别
* `$ git fetch origin develop`：只是拉取远程的develop分支的最新提交版本，到本地的`.git/refs/remotes` 目录【远程版本库】中，并没有合并到本地分支，也就是本地分支仍然只有`master`，没有`develop`.【本地版本库目录：`.git/refs/heads`】
* `$ git pull origin develop`：相当于：`git fetch origin develop`+`git merge develop`; 获取远程develop分支，并将其合并到当前你所在的本地分支。
* 将本地当前分支推送到远程仓库的master分支
    ```
    git push origin master
    ```
* 将dev分支推送到远程仓库的master分支
    ```
    git push origin master:dev
    ```

# Commit 的简写
```
git commit -am "something"
git push
```
相当于
```
git add .
git commit -m "something"
git push
```

## 区分 merge 和 rebase
将 <from> 的内容整合过来，在当前分支生成一个新的commit
```
git merge <from>
```
将当前分支的内容挂在<to>分支之上，相当于在<to>分支上长了一个新的commit。[如果当前分支在<to>分支的继承链中，直接当前分支的指针前移]
```
git rebase <to>
```

## 拉取远程git项目的某个分支
* master分支：默认会拉取的分支，直接`git clone URL`就可以获取到。
  ```
  $ git clone https://git.oschina.net/androidJP/Demo.git
  ```
* 非master分支：需要`git fetch`和`git checkout -b XXX remote仓库/XXX`结合使用。
  ```
  $ git clone https://git.oschina.net/androidJP/Demo.git
  ///项目的默认分支被克隆下来了
  $ git branch
  /// 输出：如master 等已经拉取到的分支名和当前分支
  $ git fetch
  $ git checkout -b develop origin/develop
  /// 拉取远程的develop分支并本地切换到了develop分支
  ```

## 远程分支pull不了
原因：你的本地分支没有关联上远程分支。
* 方法一：直接指定要pull哪个远程分支（默认会merge）：
  ```
  $ git pull origin <远程分支名>
  ```
* 方法二：绑定某个分支，之后就直接`git pull` 即可：
  ```
  $ git branch --set-upstream-to=origin/<远程分支名>  <本地分支名>
  ```

## 本地创建一个新分支并推送到远程，让远程也生成新分支
  ```
  $ git checkout -b branchA
  ....
  $ git add . 
  $ git commit -m "what you change"
  /// 将本地分支推到远程的同名分支上
  $ git push origin branchA
  或者  (-u 的作用是选定默认远程仓库)
  $ git push -u origin branchA
  ```

## 将本地project push到自己的github/码云仓库中
无论如何，得在相应平台上事先创建一个项目
```
git init
git add .
git commit -m "first commit"
git remote add origin https://github.com/androidjp/<你的远程项目名>.git
git remote add origin https://gitee.com/androidJP/<你的远程项目名>.git
git push -u origin master
```

# 看Log
* `git log --pretty=oneline`: 查看当前分支的所有的提交记录
* `git log --oneline -3`: 查看最近三次提交
* `git log --graph --decorate --oneline --all`: 查看所有分支的所有提交记录树状图
* `git log --graph --decorate --oneline --simplify-by-decoration A B C`: 查看A,B,C三个分支的树状图

> 说明：
> * `--decorate` 标记会让git log显示每个commit的引用(如:分支、tag等) 
> * `--oneline` 一行显示
> * `--simplify-by-decoration` 只显示被branch或tag引用的commit
> * `--all` 表示显示所有的branch

# 回滚的艺术
## 版本回滚和还原
1. 回滚
   * 法一：通过`git log`查看版本号，然后通过版本号来定位`HEAD`应该指向哪里。
     ```
     $ git log --pretty=oneline
     $ git reset --hard 12345
     ```
   * 法二：通过`HEAD^` 等，回滚到上一个/上N个版本
     ```
     // 上个版本
     $ git reset --hard HEAD^
     //上上个版本
     $ git reset --hard HEAD^^
     // 前100个版本
     $ git reset --hard HEAD~100
     ```
2. 还原
    ```
    $ git reflog  // 可查看历史操作记录，最新的记录在第一条
    $ git reset --hard <版本号>
    ```
   原理：还是使用`reset --hard <版本号>`的方式实现，并通过`git reflog`来获取以前的git操作记录，从而得到对应的版本号。

## 不小心写了句“老板真欠扁”，怎么办？
* 【修改处于工作区】幸好，我的文件还在工作区，没有被add。
   ```
   ///将我这个文件myFile.txt的所有修改都清空
   $ git checkout -- myFile.txt
   ```
* 【修改处理暂存区】哦哦，我的文件add进了暂存区。
   ```
   // 第一步，还原 add 这个操作（暂存区 --> 工作区）
   $ git reset HEAD myFile.txt
   // 第二步，清空工作区（工作区 --> 未修改）
   $ git checkout -- myFile.txt
   ```
* 【修改已提交】如果你已经commit了
  恭喜你，只能用版本回滚了。（如果老板没有你最新提交版本的版本号的话）
  * 法一：`git rebase -i head~<N>` 把包含你的错误提交在内的前N的提交，重新选择，其中，选择`drop`给你的错误提交，这样，就删除了你的错误提交了。
  * 法二：`git reset --hard HEAD~1` 把你骂老板的本地提交强行回滚到之前的一个提交，虽然这样本地依然存在这样的一个记录，但是当你push时，这个记录是不会推送到远程仓库里面的。
* 【修改已push】如果已经push code了，那怎么办呢？****理论上，是无论如何，骂老板的记录都会在的。
  * 法一：`git revert <错误提交id>` 把错误纠正，然后提交，这么做，这棵commit树上的记录中，你的错误提交记录依然存在，能被人看到。（所以，在这种case下，不推荐）
  * 法二：`git rebase -i head~<N>`，这种方式，在这种情况下，类似revert，对远程分支`origin/master`是不会达到删除提交记录的效果的，类似于和本地`master`分支做merge/rebase，最终和revert的效果一致
      ```
      git rebase -i head~<N> // 只选择正确的commit
      ....
      git merge / git rebase
      git push
      ```
## git reset 详解
```
git reset [--mixed | --soft | --hard | --merge | --keep] [-q] [<commit>]
```
* `--mixed`：【默认】回退版本库，暂存区。
* `--hard`：回退版本库，暂存区，工作区
* `--soft`：回退版本库

### --keep例子
假设你正在编辑一些文件，并且已经提交，接着继续工作，但是现在你发现当前在working tree中的内容应该属于另一个branch，与这之前的commit没有什么关系。此时，你可以开启一个新的branch，并且保留着working tree中的内容。 
```
$ git tag start 
$ git checkout -b branch1 
$ edit 
$ git commit ...                            (1) 
$ edit 
$ git checkout -b branch2                   (2) 
$ git reset --keep start                    (3)
```
* (1) 这次是把在branch1中的改变提交了。 
* (2) 此时发现，之前的提交不属于这个branch，此时你新建了branch2，并切换到了branch2上。 
* (3) 此时你可以用reset --keep把在start之后的commit清除掉，但是保持working tree不变。 

### --merge例子
在被污染的working tree中回滚merge或者pull 
```
$ git pull                         (1) 
Auto-merging nitfol 
Merge made by recursive. 
nitfol                |   20 +++++---- 
... 
$ git reset --merge ORIG_HEAD      (2)
```
* (1) 即便你已经在本地更改了一些你的working tree，你也可安全的git pull，前提是你知道将要pull的内容不会覆盖你的working tree中的内容。 
* (2) git pull完后，你发现这次pull下来的修改不满意，想要回滚到pull之前的状态，从前面的介绍知道，我们可以执行`git reset --hard ORIG_HEAD`，但是这个命令有个副作用就是清空你的working tree，即丢弃你的本地未add的那些改变。为了避免丢弃working tree中的内容，可以使用`git reset --merge ORIG_HEAD`，注意其中的`--hard`换成了`--merge`，这样就可以避免在回滚时清除working tree。 

# 关于HEAD
一整个分支树上，同时只有一个HEAD指针。
## 基础
### 1. 看 HEAD 指向
看 HEAD 指向，可以通过 `cat .git/HEAD` 查看， 如果 HEAD 指向的是一个引用，还可以用 `git symbolic-ref HEAD` 查看它的指向
### 2. 让 HEAD 指向某个指定的commit，而不是某个分支
```
git chekout <commit id>
```

## 让master分支强行指向dev分支的第3个的commit
```
git checkout dev // 首先让master分支的HEAD指向对应的分支dev
git branch -f master HEAD~3
```
## 如何在几个提交记录间随意穿梭
```
/// 查看logId
git log --pretty=oneline

// 选择logID来跳转
git reset --hard <logId>
```

# 用git快速debug
* `git bisect start [终点] [起点]`: 开始检查某一段提交中的哪一次提交出了问题，每次都用二分法，看中间的这次提交有没有问题，一般：`git bisect start HEAD 4d83cf`
* `git bisect good`: 如果当前的提交是没问题的，那么就标记为good
* `git bisect bad`: 如果当前的提交是有问题的，那么就标记为bad
* `git bisect reset`: 推出差错过程，指针回到HEAD

# cherry-pick 和 rebase -i
## 抽取想要的几个commit
`git cherry-pick <logId[空格..]>`: 选择某个分支，merge到本分支。
## 如何合并历史的几个commit到一个commit
这里有个例子：
```
192eb39de1fdbb56f1f2693c99611365855d58d0 (HEAD -> master) xx
f0819adfb5e1bed8bc2fa86d1ce3f6ee878c6f0c add `333`
8f26c03c69a06928b6045087e0b01b6ac88a106a add `222`
01579e444c53c20f1ef3bfa18a0832f1ae5b26cd add `111`
436c7d3166d95bd1a0626ea660fec9c50714441d init abc.txt & add `abc`
```
现在我想变成：
```
352f0d9fe36c0e38938be8cb3a881f93f8158f18 (HEAD -> master) add `111` -> add `222` -> add `333` -> xx
436c7d3166d95bd1a0626ea660fec9c50714441d init abc.txt & add `abc`
```
1. `git rebase -i HEAD~4`: 首先基于`HEAD~4`这个父commit，选择最顶端4个commit
2. 会弹出一个交互框，内部可以看到`pick`, `squash`等字样，我们会这样写：
    ```
    pick commit_1 111
    squash commit_2 222
    squash commit_3 333
    squash commit_4 xx
    ```
    这里是想把前3个子commit合并到前面的第一个父commit中。
3. `git rebase --continue`: 填写一波description并生成一个新的commit，覆盖原来的4个子commit。

# 和远程仓库打交道
1. 创建
   ```
   $ git remote add origin git@github.com:michaelliao/learngit.git
   ```
2. 推送
   ```
   /// 第一次推送，-u 作用是将本地仓库与远程仓库origin 绑定
   $ git push -u origin master 

   /// 以后的推送，则不需要再次绑定了
   $ git push origin master
   ```
3. 查看远程库信息
   ```
   $ git remote  /// 远程库名
   $ git remote -v ///详细信息
   ```
4. 查看远程分支
   ```
   $ git branch -a  ///查看所有分支（包括本地和远程仓库）
   $ git branch 
   ```
4. 删除远程分支
   ```
   $ git branch -r -d origin/develop    /// 删除远程分支develop
   $ git push origin :develop   ///删除远程的develop分支
   ```
## 分支重命名
```
git branch [<options>] (-m | -M) [<old-branch>] <new-branch>
```
例子：将`feature/canvas`重命名为：`feature/canvas2`
```
git branch -M feature/canvas feature/canvas2
```

## 让本地分支追溯到远程某个分支
---
如果发现，自己创建的本地分支pull不了，那么，可能是本地分支没有与远程分支建立关联，这时：
```
git branch --set-upstream-to=origin/<branch> develop
```
就可以建立连接，之后，就是各种push、pull和merge了！

## Pull 不下来怎么办？
---
* 如果发现是这种报错：`fatal: refusing to merge unrelated histories`【在新建Ionic项目的时候经常这样】，那么，用这条语句来pull:
   ```
   git pull origin master --allow-unrelated-histories
   ```
## git clone 指定目录
---
如果我们不想每次先cd到那个目录在进行clone操作，那么，这句命令很有用：
```
git clone <git url> "C:\a\b"
```
其中：`a`表示指定目录，`b`表示你自定义的文件夹名，如：
```
>git clone https://github.com/androidjp/xxxxx.git "D:\aaa\bbb"
```
最终，会创建`aaa\bbb`目录和文件夹，然后在内部拉取所有代码。

# stash
## dev分支码到一半，发现master分支的版本有bug，要马上改
**思路：**dev的修改先保存起来，然后切换到master分支，再打bug分支修复bug，修复成功并合并后，最终切回dev分支，并取回之前保存的dev分支的修改内容，继续码。
**步骤**：
1. 暂存dev工作现场
   ```
   $ git stash
   ```
2. 切回master, 打bug修复分支，修复并合并
   ```
   $ git checkout master
   $ git checkout -b bug-solve-101
   ////修复bug中。。。。
   $ git checkout master
   $ git merge --no-ff -m "bug 101修复成功" bug-solve-101
   ```
3. 最终，切回dev分支，并还原工作现场
   ```
   $ git stash pop   /// 还原现场，并清除存储栈中的内容
   /// $ git stash list /// 查看存储栈中的工作现场列表
   /// $ git stash apply  ///只恢复，不删除栈
   /// $ git stash drop /// 删除栈存储区
   ```

# Lag标签
1. `git tag <name>`：新建一个标签，默认为HEAD，也可以指定一个commit id；
2. `git tag -a <tagname> -m "blablabla..."`：可以指定标签信息；
3. `git tag -s <tagname> -m "blablabla..."`：可以用PGP签名标签；
4. `git tag`：可以查看所有标签
5. `git push origin <tagname>`：可以推送一个本地标签；
6. `git push origin --tags`：可以推送全部未推送过的本地标签；
7. `git tag -d <tagname>`：可以删除一个本地标签；
8. `git push origin :refs/tags/<tagname>`：可以删除一个远程标签。


# 解决每次拉取、提交代码时都需要输入用户名和密码
1.在~/.gitconfig目录下多出一个文件，用来记录你的密码和帐号
```
git config --global credential.helper store
```
2. 再最后输入一次正确的用户名和密码，就可以成功的记录下来

# git 使用 push 提交到远程仓库出现 The requested URL returned error: 403 错误
## 问题描述
电脑已经注册过一个 github 帐号，一直在本机使用，配置过 SSH。

新建另一个 github 帐号，本地建立好项目之后，使用命令：$ git push -u origin master 时出现以下错误：
```
remote: Permission to userName/repositorieName.git denied to clxering.
fatal: unable to access 'https://github.com/userName/repositorieName.git/': The requested URL returned error: 403
```

## 问题原因
问题主要出在原注册账号上，系统保存了账号的信息。在使用新帐号时，信息不一致，所以报错。

## 解决方案
1. 打开cmd，输入命令：`rundll32.exe keymgr.dll,KRShowKeyMgr`，出现系统存储的用户名和密码窗口；
2. 将 github 相关的条目删除；
3. 重新执行命令：`$ git push -u origin master`，提示输入账户名及密码即可。

# 查看git信息存储位置
```
git help -a | grep credential
```
查看自己系统支持的crendential, cache 代表内存中的缓存，store 代表磁盘。

# 查看cache、store等git配置信息
```
git config credential.helper
```
命令可以看到 cache、store、osxkeychain(钥匙串)中是否还有git的配置信息

# 配置全局用户名和邮箱
一般配置方法：
```
git config --global (--replace-all) user.name "你的用户名"
git config --global (--replace-all) user.email "你的邮箱"
```
如果上述步骤没有效果，我们就需要清除缓存(`.gitconfig`)
```
git config --local --unset credential.helper
git config --global --unset credential.helper
git config --system --unset credential.helper
```

# git revert 一次 revert 多个 commit
## 场景
假如git commit 链是
```
A -> B -> C -> D
```
如果想把B，C，D都给revert，除了一个一个revert之外，还可以使用range revert
```
git revert B^..D 
```
这样就把B,C,D都给revert了，变成：
```
A-> B ->C -> D -> D'-> C' -> B'
```
## 用法
`git revert OLDER_COMMIT^..NEWER_COMMIT`

如果我们想把这三个revert不自动生成三个新的commit，而是用一个commit完成，可以这样：
```
git revert -n OLDER_COMMIT^..NEWER_COMMIT
git commit -m "revert OLDER_COMMIT to NEWER_COMMIT"
```