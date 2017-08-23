---
title: git常用命令总结
date: 2017-07-02 01:27:21
categories: Git
tags:
  - 工具
---

### git常用命令总结
- `git add filename`
- `git commit -m "some info"`
- `git rm --cached <filename>`
  用于`git add`后进行撤销
- `git commit --amend --no-edit`
  应用场景: 上一次commit后遗忘了某些东西, 现在修改后还想提交到上一次的commit中, 不然只使用`git commit -m "test"`会新增一条commit记录
  但是`git reflog`和`git log --oneline`显示的会不一样, `git reflog`会显示一条类似amend的记录
  ```
  ql@ubuntu:~/toturial$ git log --oneline 
  15c1c20 change 2
  2015298 change 1
  0a191b4 第一次提交
  ql@ubuntu:~/toturial$ 
  ql@ubuntu:~/toturial$ git reflog 
  7c0ec01 HEAD@{1}: commit (amend): change 2
  5b4059c HEAD@{2}: commit: change 2
  2015298 HEAD@{3}: commit: change 1
  0a191b4 HEAD@{4}: commit (initial): 第一次提交
  ql@ubuntu:~/toturial$ 
  ql@ubuntu:~/toturial$ 
  ```
  使用该命令后change的id是会变的

- `git status -s`
  缩写形式查看状态
- `git diff`
  查看**工作**区与上一次提交的不同, 如果add后是不能显示的
- `git diff --cached`
  查看**暂存区**与上一次提交的不同, 即查看add后的不同
- `git diff HEAD`
  查看**工作区**和**暂存区**与**上一次提交**的所有不同
- `git log --oneline`
  查看commit的日志
  ```
  ql@ubuntu:~/toturial$ git log --oneline
  5b4059c change 2
  2015298 change 1
  0a191b4 第一次提交
  ```
- `git reflog`
  查看总的提交情况
  ```
  ql@ubuntu:~/toturial$ git reflog
  5b4059c HEAD@{0}: commit: change 2
  2015298 HEAD@{1}: commit: change 1
  0a191b4 HEAD@{2}: commit (initial): 第一次提交
  ql@ubuntu:~/toturial$ 
  ```
- `git reset --hard HEAD`
  回到上一个版本
- `git reset --hard 版本号`
  该命令可以回到之前的任意版本, HEAD命令只能回到最近的一次版本
- `git checkout 版本号 -- 文件名`
  将文件恢复到指定版本(一般指单个文件)
  ```
  ql@ubuntu:~/toturial$ git log --oneline 
  15c1c20 change 2
  2015298 change 1
  0a191b4 第一次提交
  ql@ubuntu:~/toturial$ 
  ql@ubuntu:~/toturial$ 
  ql@ubuntu:~/toturial$ git checkout 15c1c20 -- 1.py
  ```
- `git checkout -- 文件名`
  恢复回之前的最新提交版本
- `git log --oneline --graph`
  ```
  ql@ubuntu:~/Test$ git log --oneline --graph 
  * bad5622 1.py
  ql@ubuntu:~/Test$
  ```

  `*`表示分支的意思

  ```
  ql@ubuntu:~/Test$ git log --oneline --graph 
  *   ae42ff2 merge info
  |\  
  | * a741b97 change 2
  |/  
  * 3448add commit 1
  ql@ubuntu:~/Test$ 
  ```

  如上, 可以看到分支合并的信息
- `git commit -am "info"`
  提交的时候自动add
  只针对于之前已经add过得文件, 如果是新文件的话还需要先执行add
- `git stash`
  比如工作突然被打断要改bug，可以新建bug分支解决完后合并到master, 之后删除bug分支再回来继续工作, 此时需要隐藏正在做的工作
  - `git stash`
  - `git stash pop`恢复的同时删除记录(等同于如下三条)
  - `git stash apply` 恢复
  - `git stash list` 列出列表
  - `git stash drop` 删除一条

<!--more-->

### 合并分支

- `git merge branch_name`
  - `git merge 分支名`直接这样使用不会看到合并信息
  - `git merge  --no-ff -m "info" 分支名`这样在log中能看到提交的信息
- dev分支和master合并发生冲突
  1. 新建分支并切换`git checkout -b 分支名`
  2. 切换分支`git checkout 分支名`
  3. 合并分支， 比如要合并dev到master上
     - 先切换到master: `git checkout master`
     - 合并`git merge dev`
     - 如果出现冲突， 手动解决后
       - git add 文件名
       - git commit -m "......
     - 此时dev分支并没有改变
       - 切换到dev分支: `git checkout dev`
       - 合并master到dev: `git merge master`, 这样才能保master和dev分支一致 ​
- git rebase合并分支
  - 使用git rebase合并分支，如果出现冲突，  会出现在另一个分支上
  - 解决冲突后， 直接add就能将解决后的版本加入master分支



https://segmentfault.com/a/1190000005370037

https://segmentfault.com/a/1190000009893041


