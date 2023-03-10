> 不要直接对公共远程仓库 clone，应先 fork 出个人远程仓库，再 clone 回本地进行作业，最后将修改提交到个人远程仓库，向公共远程仓库提交 merge request 申请

![[20221211_004638.jpg]]

## 命令

- 查看历史命令 git reflog
- 查看提交历史 git log
- 查看当前分支状态 git status
- 查看节点区别 git diff
- 创建分支 git branch/git checkout -b
- 切换分支 git checkout \[branch-name\]
- 创建节点 git init
  - 设置个人信息标识 git config --global user.name/email
- 绑定远程库 git remote add \[url.git\]
- 将工作区提交到暂存区 git add .
- 将暂存区提交到本地库 git commit -am "msg"
  - "msg"需符合提交规范
- 推送本地库到远程库 git push origin master **--force(慎用)**
- 拉取远程库到本地库，且合并本地分支 git pull
- 拉取远程库到本地库，但不合并，作为新分支 git fetch
- 克隆远程库到工作区 git clone
- 工作区内容回退 git checkoutb \<file-name\> | .
- 暂存区删除文件 git rm --cache
- **本地库回退 git reset \[commit ID\] | \[HEAD^^^\]**
- 合并分支的三种方式
  - 三方非线性合并，产生菱形路径 git merge
    - 三方线性变基 git rebase，用之前应与协作者商量
    - **合并节点 git cherry-pick \[commit ID\]**
- 冲突处理
  - 冲突是由于不同分支在同一文件同一行产生了不同提交内容
  - 产生冲突后，先商量好最终留存版本，再 commit
- git fork
- git patch

## 合并历史 commit

- 有时开发过程中提交的 commit 很多，但实际需要展示的进程节点仅需几个，因此可以将若干 commit 合并提交，保持提交历史的整洁

1. 使用 git log 查看历史提交，确定想要合并的区间
2. 使用 git rebase -i \[commitID\]将 HEAD 指针放到区间的前一个节点  
   2.1 弹出交互式界面，将需要保留的节点前缀保留为 pick，将需要合并的节点前缀改为 squash(保留该条 msg 或简写 s), 或 fixup(舍弃该条 msg, 简写 f)
   2.2 继续弹出交互式界面，添加合并后节点的 message
3. 使用 git log 验证效果

## 同步源仓库到 fork 仓

- 开发时常需要将源仓库 fork 到个人远程仓，然后再 clone 到本地仓进行开发
- 但有时 fork 的分支仓库相比于源仓库存在版本落后情况，需要将其与源仓库最新版本同步
- **思路** ：使用本地仓作为第三方中间点，同时建立到 fork 分支仓库和源仓库的连接，并在本地同步版本差异，然后 push 到 fork 分支仓库

步骤：

1. 列出当前为 Fork 配置的远程仓库，一般应只有到 fork 仓的连接

git remote -v

2. 指定将与 Fork 同步的新远程上游（upstream）仓库

git remote add upstream \[源仓库\].git

3. 验证为 Fork 指定的新上游仓库，此时本地仓同时与上游仓和 fork 仓建立连接。

git remote -v

4. 从上游仓库获取最新的分支及其各自的提交，传送到本地，对 master 分支的提交将存储在本地分支 upstream/master 中

git fetch upstream

5. 切换到本地的主分支

git checkout master

6. 将来自 upstream/master 的更改合并到本地 master 分支中。这就实现了与上游仓库的同步，而不会丢失本地的更改。注意此步可能发生冲突。

git merge upstream/master

7. 最后推送到远程 fork 仓库就完成了

git push origin master

## 同步远程 master 分支到本地仓

- git fetch origin 远程分支:tmp //在本地新建一个 temp 分支，并将远程 origin 仓库的 master 分支代码下载到本地 temp 分支
- git diff tmp //来比较本地代码与刚刚从远程下载下来的代码的区别
- git merge tmp //合并 temp 分支到本地的 master 分支

- git branch -d tmp //如果不想保留 temp 分支 可以用这步删除
