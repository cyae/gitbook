>不要直接对公共远程仓库clone，应先fork出个人远程仓库，再clone回本地进行作业，最后将修改提交到个人远程仓库，向公共远程仓库提交merge request申请

![[20221211_004638.jpg]]

## 命令
* 查看历史命令 git reflog  
* 查看提交历史 git log  
* 查看当前分支状态 git status  
* 查看节点区别 git diff  
* 创建分支 git branch/git checkout -b  
* 切换分支 git checkout \[branch-name\]  
* 创建节点 git init   
	* 设置个人信息标识 git config --global `user.name/email ` 
* 绑定远程库 git remote add \[url.git\]  
* 将工作区提交到暂存区 git add .  
* 将暂存区提交到本地库 git commit -am "msg"   
    * "msg"需符合提交规范  
* 推送本地库到远程库 git push origin master **--force(慎用)**
* 拉取远程库到本地库，且合并本地分支 git pull  
* 拉取远程库到本地库，但不合并，作为新分支 git fetch  
* 克隆远程库到工作区 git clone  
* 工作区内容回退 git checkoutb \<file-name\> | .  
* 暂存区删除文件 git rm --cache  
* **本地库回退 git reset \[commit ID\] | \[HEAD^^^\]**  
* 合并分支的三种方式   
	* 三方非线性合并，产生菱形路径 git merge  
    * 三方线性变基 git rebase，用之前应与协作者商量  
    * **合并节点 git cherry-pick \[commit ID\]**  
* 冲突处理   
	* 冲突是由于不同分支在同一文件同一行产生了不同提交内容  
	* 产生冲突后，先商量好最终留存版本，再commit  
* git fork  
* git patch  

## 合并历史commit  
  
* 有时开发过程中提交的commit很多，但实际需要展示的进程节点仅需几个，因此可以将若干commit合并提交，保持提交历史的整洁  
    1. 使用git log查看历史提交，确定想要合并的区间  
    2. 使用git rebase -i [commitID]将HEAD指针放到区间的前一个节点  
	    2.1 弹出交互式界面，将需要保留的节点前缀保留为pick，将需要合并的节点前缀改为squash(保留该条msg或简写s),  或fixup(舍弃该条msg, 简写f)
	    2.2 继续弹出交互式界面，添加合并后节点的message  
    3. 使用git log验证效果

## 同步源仓库到fork仓

* 开发时常需要将源仓库fork到个人远程仓，然后再clone到本地仓进行开发  
* 但有时fork的分支仓库相比于源仓库存在版本落后情况，需要将其与源仓库最新版本同步  
* **思路** ：使用本地仓作为第三方中间点，同时建立到fork分支仓库和源仓库的连接，并在本地同步版本差异，然后push到fork分支仓库

<p class="infinite-list-item"><strong>步骤：</strong></p>
<p class="infinite-list-item">1. 列出当前为 Fork 配置的远程仓库，一般应只有到fork仓的连接 </p>
<p class="infinite-list-item">git remote -v</p>
<p class="infinite-list-item">2. 指定将与 Fork 同步的新远程上游（upstream）仓库</p>
<p class="infinite-list-item">git remote add upstream [源仓库].git</p>
<p class="infinite-list-item">3. 验证为 Fork 指定的新上游仓库，此时本地仓同时与上游仓和fork仓建立连接。</p>
<p class="infinite-list-item">git remote -v</p>
<p class="infinite-list-item">4. 从上游仓库获取最新的分支及其各自的提交，传送到本地，对 master 分支的提交将存储在本地分支 upstream/master 中</p>
<p class="infinite-list-item">git fetch upstream</p>
<p class="infinite-list-item">5. 切换到本地的主分支</p>
<p class="infinite-list-item">git checkout master</p>
<p class="infinite-list-item">6. 将来自 upstream/master 的更改合并到本地 master 分支中。这就实现了与上游仓库的同步，而不会丢失本地的更改。注意此步可能发生冲突。</p>
<p class="infinite-list-item">git merge upstream/master</p>
<p class="infinite-list-item">7. 最后推送到远程fork仓库就完成了</p>
<p class="infinite-list-item">git push origin master</p>
<p class="infinite-list-item"><strong>同步远程master分支到本地仓</strong></p>
<ul>
<li>git fetch
     origin 远程分支:tmp<br />
     //在本地新建一个temp分支，并将远程origin仓库的master分支代码下载到本地temp分支</li>
<li>git diff
     tmp<br />
     //来比较本地代码与刚刚从远程下载下来的代码的区别</li>
<li>git merge
     tmp<br />
     //合并temp分支到本地的master分支</li>


</ul>



git branch -d tmp<br />
//如果不想保留temp分支 可以用这步删除