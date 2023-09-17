
## 合并与变基

关于 git rebase ，首先要理解的是它解决了和 git merge 同样的问题。这两个命令都旨在将更改从一个分支合并到另一个分支，但二者的合并方式却有很大的不同。

当你在专用分支上开发新 feature 时，然后另一个团队成员在 master 分支提交了新的 commits，这会发生什么？这会导致分叉的历史记录，对于这个问题，使用 Git 作为协作工具的任何人来说都应该很熟悉。

![](Pasted%20image%2020230917224217.png)

现在，假设在 master 分支上的新提交与你正在开发的 feature 相关。需要将新提交合并到你的 feature 分支中，你可以有两个选择：merge 或者 rebase。

## Merge 方式

最简单的方式是通过以下命令将 master 分支合并到 feature 分支中：

```shell
git checkout feature
git merge master
```

或者，你可以将其浓缩为一行命令：
```shell
git merge feature master
```

这会在 feature 分支中创建一个新的 merge commit，它将两个分支的历史联系在一起，请看如下所示的分支结构：
![](Pasted%20image%2020230917224434.png)

使用 merge 是很好的方式，因为它是一种 **非破坏性** 操作。现有分支不会以任何方式被更改。这避免了 rebase 操作所产生的潜在缺陷（下面讨论）。

另一方面，这也意味着 feature 分支每次需要合并上游更改时，它都将产生一个**额外**的合并提交。如果master 提交非常活跃，这可能会严重污染你的 feature 分支历史记录。尽管可以使用高级选项 git log 缓解此问题，但它可能使其他开发人员难以理解项目的历史记录

## Rebase 方式
作为 merge 的替代方法，你可以使用以下命令将 master 分支合并到 feature分支上：

```shell
git checkout feature
git rebase master
```

这会将整个 feature 分支移动到 master 分支的顶端，从而有效地整合了所有 master 分支上的提交。但是，与 merge 提交方式不同，rebase 通过为原始分支中的**每个提交**创建全新的 commits 来 **重写** 项目历史记录。

![](Pasted%20image%2020230917224649.png)

rebase 的主要好处是可以获得更清晰的项目历史。首先，它消除了 git merge 所需的不必要的合并提交；其次，正如你在上图中所看到的，rebase 会产生完美线性的项目历史记录，你可以在 feature分支上没有任何分叉的情况下一直追寻到项目的初始提交。这样可以通过命令 git log，git bisect 和 gitk 更容易导航查看项目。

但是，针对这样的提交历史我们需要权衡其「安全性」和「可追溯性」。如果你不遵循 [Rebase 的黄金法则](## Rebase 的黄金法则)，重写项目历史记录可能会对你的协作工作流程造成灾难性后果。而且，rebase 会丢失合并提交的上下文， 你也就无法看到上游更改是何时合并到 feature 中的。

## 交互式 Rebase

交互式 rebase 使你有机会在将 commits 移动到新分支时更改这些 commits。这比自动 rebase 更强大，因为它提供了对分支提交历史的完全控制。通常，这用于在合并 feature 分支到 master 之前清理其杂乱的历史记录。

要使用交互式 rebase，需要使用 git rebase 和 -i 选项：

```shell
git checkout feature
git rebase -i master
```

这将打开一个文本编辑器，列出即将移动的所有提交：
```shell
pick 33d5b7a Message for commit #1
pick 9480b3d Message for commit #2
pick 5c67e61 Message for commit #3
```

此列表准确定义了执行 rebase 后分支的外观。通过更改 pick 命令或重新排序条目，你可以使分支的历史记录看起来像你想要的任何内容。例如，如果第二次提交 fix 了第一次提交中的一个小问题，您可以使用以下 fixup 命令将它们浓缩为一个提交：

```shell
pick 33d5b7a Message for commit #1
fixup 9480b3d Message for commit #2
pick 5c67e61 Message for commit #3
```

保存并关闭文件时，Git将根据您的指示执行 rebase，从而产生如下所示的项目历史记录：
![](Pasted%20image%2020230917230556.png)


消除这种无意义的提交使你的功能历史更容易理解。这是 git merge 根本无法做到的事情。至于 commits 条目前的 pick、fixup、squash 等命令，在 git 目录执行 git rebase -i 即可查看到，大家按需重排或合并提交即可，注释说明非常清晰，在此不做过多说明:

## Rebase 的黄金法则
一旦你理解了什么是 rebase，最重要的是要学习什么时候不能使用它。git rebase 的黄金法则是**永远不要在公共分支上使用它**。

例如，如果你 rebase master 分支到 feature 分支之上会发生什么：
![](Pasted%20image%2020230917230742.png)

rebase 将所有 master 分支上的提交移动 feature 分支的顶端。问题是这只发生在 你自己 的存储库中。所有其他开发人员仍在使用原始版本的 master。由于 rebase 导致全新 commit，Git 会认为你的 master 分支历史与其他人的历史不同。

此时，同步两个 master 分支的唯一方法是将它们合并在一起，但是这样会产生额外的合并提交和两组包含相同更改的提交（原始提交和通过 rebase 更改的分支提交）。不用说，这是一个令人非常困惑的情况。

因此，在你运行 git rebase 命令之前，总是问自己，还有其他人在用这个分支吗？ 如果答案是肯定的，那就把你的手从键盘上移开，开始考虑采用非破坏性的方式进行改变（例如，git revert 命令）。否则，你可以随心所欲地重写历史记录。

### Force Push

如果你违背黄金准则，尝试将 rebase 了的 master 分支推送回 remote repository，Git 将阻止你这样做，因为它会与远程master 分支冲突。但是，你可以通过传递 --force 标志来强制推送，如下所示：
```shell
# Be very careful with this command
git push --force
```

这样你自己 repository 的内容将覆盖远程 master分支的内容，但这会使团队的其他成员感到困惑。因此，只有在确切知道自己在做什么时才要非常小心地使用此命令。

如果没有人在 feature branch 上作出更改，你可以使用 force push 将本地内容推送到 remote repository 做清理工作


## 本地清理

将 rebase 纳入工作流程的最佳方法之一是清理本地正在进行的功能。通过定期执行交互式 rebase，你可以确保功能中的每个提交都具有针对性和意义。这可以使你在编写代码时无需担心将其分解为隔离的提交(多个提交)，你可以在事后修复整合它。

使用 git rebase 时，有两种情况：feature 父分支（例如 master ）的提交，或在 feature 中的早期提交。我们在 交互式 Rebase 部分已经介绍了第一种情况的示例。我们来看后一种情况，当你只需要修复最后几次提交时，以下命令仅做最后 3 次提交的交互式 rebase。

```shell
git checkout feature
git rebase -i HEAD~3
```

通过指定 HEAD~3 ，你实际上并没有移动分支，你只是交互式地重写其后的3个提交。请注意，这不会将上游更改合并到 feature 分支中。

![](Pasted%20image%2020230917231518.png)

如果要使用此方法重写整个功能，git merge-base 命令可用于查找 feature 分支的原始 base。以下内容返回原始 base 的提交ID，然后你可以将其传递给 git rebase：

git merge-base feature master
交互式 rebase 的使用是引入git rebase 工作流的好方法，因为它只影响本地分支。其他开发人员唯一能看到的就是你提交的最终版，这应该是一个简洁易懂易跟踪的分支历史记录。

但同样，这仅适用于 私有 feature分支。如果你通过相同的功能分支(公共分支)与其他开发人员协作，那么你是 不被允许 重写其历史记录的。

将上游更改合并到功能分支中
在 概念概述 部分中，我们了解了 feature 分支可以使用 git merge 或 git rebase 合并 master 分支的上游更改 。merge 是一个安全的方式，可以保留存 git repository 的整个历史记录，而 rebase 则是通过将 feature 分支移动到 master 顶端来创建线性历史记录。

这种使用 git rebase 类似于本地清理，但在此过程中它包含了那些来自 master 上游提交。

请记住，将当前提交 rebase 到远程 branch(非 master 分支)一样是合法的。当与另一个开发人员协作使用相同的功能并且你需要将他们的更改合并到你的 repository 时，就会发生这种情况。

例如，如果你和另一个名为 John 的开发人员添加了对 feature 分支的提交，在你 fetch (注意 fetch 并不会自动 merge )来自 John 的远程 feature分支后，你的 repository 可能如下所示：

在同一个功能分支上进行协作

你可以整合上来自上游的分叉：要么用 john/feature merge 本地 feature ，要么 rebase 本地feature 到john/feature 的顶部。

合并与重新定位到远程分支

请注意，此 rebase 不违反 Rebase 黄金规则，因为只有你的本地 feature 提交被移动， 之前的所有内容都不会受到影响。这就像是说 "将我的更改添加到 John 已经完成的工作中"。在大多数情况下，这比通过合并提交与远程分支同步更直观。

默认情况下，使用 git pull 命令执行合并，但你可以通过向其传递 --rebase 选项来强制它将远程分支 以 rebase 方式集成。

git pull --rebase
使用 Pull 请求 Review Feature
如果你在代码审查过程中使用 pull 请求，在使用了 pull 请求之后你应该避免使用 git rebase 。一旦你发出 pull 请求，其他开发人员就会查看你的提交，这意味着它是一个 公共 分支。重写其历史记录将使 Git 和你的队友无法跟踪添加到该功能的任何后续提交。

其他开发人员的任何更改都需要合并 git merge 而不是 git rebase。

因此，在提交拉取请求之前，通常使用交互式 rebase 清理代码通常是个好的办法。注意使用顺序

集成已批准的功能
在你的团队批准某项 feature 后，你可以选择将该功能 rebase 到 master 分支的顶端，然后git merge再将该功能集成到主代码库中。

这与将上游更改合并到 feature 分支中的情况类似，但由于你不允许在 master 分支中重写提交，因此你必须最终使用 git merge 该功能进行集成。但是，通过在 merge 之前执行 rebase，你可以确保会以 fast-forward 方式 merge，从而产生完美的线性历史记录。

使用和不使用rebase将功能集成到master中

如果您不熟悉 git rebase，可以随时在临时分支中执行 rebase。这样，如果你不小心弄乱了功能的历史记录，可以查看原始分支，然后重试。例如：

git checkout feature
git checkout -b temporary-branch
git rebase -i master

# [Clean up the history]

git checkout master
git merge temporary-branch
总结
如果你更喜欢没有不必要的干净的合并提交，线性历史记录，你就需要开始了解使用 rebase 功能。同时你应该会使用 git rebase 而不是 git merge 集成来自另一个分支的更改。

另一方面，如果你想保留项目的完整历史记录并避免重写公共提交的风险，你可以坚持下去git merge。这两种选择都是完全有效的，但至少现在你可以选择利用 git rebase 的好处 。

灵魂追问
你有使用过 git rebase 吗？这样清晰的线形历史是不是可以尝试一下呢？
交互式 rebase 提交条目前的命令 fixup 等你能灵活使用吗
在 feature 分支上开发时，试试 git pull -rebase?
