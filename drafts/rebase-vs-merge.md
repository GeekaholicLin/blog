### 分支合并

分支说明：`master`表示基础分支，`bugfix`表示从`master`分出的分支。

`git merge`以及`git rebase`都可以用于合并分支，但两者合并后分支的历史记录会有很大的差别。

`git pull`相当于`git fetch + git merge --no-ff`，`git pull --rebase`相当于`git fetch + git rebase`。`merge`是`git pull`的默认行为，可以通过配置改变某个分支的 pull 行为`git config branch.master.rebase true`，或者通过配置改变全部分支的行为`git config branch.autoSetupRebase true`。

```bash
git pull origin master

# 等价于
git fetch origin master # 获取最新记录
git merge origin/master # 合并
```

`merge`的合并策略有两种，一种是快进合并的`fast-forward`，当`bugfix`合并到`master`的时候，直接将`master`分支的 HEAD 指向`bugfix`最新 commit 即可。而如果合并的时候`master`有了其他 commit，也就是出现了分叉，就会合并两个分支到一个新的 commit 用于汇合两者。而`git merge bugfix --no-ff`可以取消`fast-forward`，把`fast-forward`也当成有分叉的情况，产生新的 commit。

这里说一下题外话，当可以`fast-forward`的时候进行`merge`，可以使用`git merge --squash`，将多出的所有 commit 压缩成一个 commit。

`rebase`的操作的分支操作与`merge`不同，`merge`是在`master`分支上操作，而`rebase`是在`bugfix`分支上。从语义上来讲就是，`rebase`是将当前分支`bugfix`相对于目标分支`master`产生的修改（从 HEAD 到第一个公共父节点），**一个一个地**从`bugfix`移到`master`，改变它的 base，**当然，这些移动的 commit 算新的 commit**。这会使得历史记录从分叉的图结构变成了一条直线的线性结构。（注意这时候的`master`指向依然是之前所在的 commit，需要进一步地`merge`）

```bash
git rebase bugfix # bugfix 是 <target_branch>
```

不仅仅是历史记录不同，因为操作上的差异，使得解决冲突上也有所差异。`merge`产生冲突的时候，是一次性解决，然后`git add`。而在`rebase`的过程中，Git 会停止`rebase`并让用户去解决冲突，解决完冲突后用`git add`命令去更新这些内容，然后不用执行`git-commit`, 直接执行`git rebase --continue`, 这样 git 会继续 apply 余下的补丁。在任何时候，都可以用`git rebase --abort`参数来终止`rebase`回到`rebase`开始前的状态。

所以，使用`git pull --rebase`比直接`git pull`容易导致冲突的产生，如果预期冲突比较多的话，建议还是直接`git pull`

如果想简化历史记录，可以从`master`分支更新`bugfix`，可以使用`rebase`。如果从`bugfix`更新`master`
，可以先在`bugfix`分支使用`rebase`简化提交历史记录，再在`master`分支使用`merge`更新`master`

#### rebase 的危险操作

rebase 的风险在于，会改写 commit 历史，如果操作不当会是一种破坏性的操作。一个原则就是：**禁止对一个多人协作的分支进行 rebase**，也就是说**不要**`git checkout master`切换到`master`分支（`master`分支为多人协作的分支），然后`git rebase bugfix`，将`master`的历史提交一一应用到`bugfix`。所以应该只使用 rebase 来清理你的本地工作，千万不要对那些已经被发布的提交进行这个操作。

#### rebase 的其他用途

#### 一种最佳实践

```bash
git checkout bugfix # 切换到工作分支
# 假设上游分支有更新
git pull --rebase origin bugfix # 注意，本地工作目录需要是 clean 的状态。
git rebase master # 将 bugfix 新提交的 commit 一个个应用到 master 分支
git checkout master # 切换到 master
git merge --no-ff bugfix # 强行使用 --no-ff 进行提交，使得 bugfix 合并进去的有突出来的效果，而不是一条直线，方便后续定位
git push origin master # 推送
```

### Git 中的对象

- tree：目录对象，内部包含目录和文件
- blob：文件对象，表示一个文件
- commit：存储提交时的信息，比如所在的 tree、作者、message、提交时间、当前拥有的文件等，类似一个快照

### 特殊符号

`~`和`^`是 Git 中常见的特殊符号，两者可以单独使用也可以任意搭配使用

- `~`：很多情况下使用，用于将指向回退到某次提交
- `^`：常用于有合并分支的情况，因为这种情况下会拥有多个父提交（即本次提交的上次提交），用于回退到某个被合并的分支

![特殊符号](http://image.geekaholic.cn/20191024161059.png@0.8)

比如以下 [例子](https://stackoverflow.com/questions/2221658/whats-the-difference-between-head-and-head-in-git/2222920#2222920)：

```txt
// 从上往下表示提交顺序，HEAD 目前在 A 处
// B 和 C 都是 A 的父提交，且父提交的顺序为从左到右
G   H   I   J
 \ /     \ /
  D   E   F
   \  |  / \
    \ | /   |
     \|/    |
      B     C
       \   /
        \ /
         A

A =      = A^0
B = A^   = A^1     = A~1
C = A^2
D = A^^  = A^1^1   = A~2
E = B^2  = A^^2
F = B^3  = A^^3
G = A^^^ = A^1^1^1 = A~3
H = D^2  = B^^2    = A^^^2  = A~2^2 // 先回退 2 个 commit，然后第 2 个分支
I = F^   = B^3^    = A^^3^
J = F^2  = B^3^2   = A^^3^2
```
### git stash

stash 是临时保存文件修改内容的区域。stash 可以暂时保存工作树和索引里还没提交的修改内容，您可以事后再取出暂存的修改，应用到原先的分支或其他的分支上。

还未提交的修改内容以及新添加的文件，留在索引区域或工作树的情况下切换到其他的分支时，修改内容会从原来的分支移动到目标分支。但是如果在 checkout 的目标分支中相同的文件也有修改，checkout 会失败的。这时要么先提交修改内容，要么用 stash 暂时保存修改内容后再 checkout。

### git flow

- master：只负责管理发布的状态。在提交时使用 tag 记录发布版本号
- develop：针对发布的日常开发分支，也叫「集成分支」，可以用于`nightly build`
- feature：对应的功能分支。
- hotfix：紧急修复分支
- release：预发布分支

![相关分支](http://image.geekaholic.cn/20191024164339.png@0.8)

从上图可以看到，hotfix 分支是针对紧急上线的，不应该从日常开发中的 develop 切出，而应该是线上版本的 master 切出。测试以及测试中的 bug 修复也是在 hotfix 上进行测试。上线 hotfix 完成之后，需要合并进入 develop 以及 master。

feature 是迭代或者说单个功能的开发分支，应该从 develop 中切出或更新（因为有可能多个人开发多个 feature，可命名为`feature-*`）。开发完成后，合并回 develop。

当 feature 开发、测试、bug 修复完成并合并到 develop 之后，从最新的 develop 切出一个`release-*`分支用于预发布（相当于 staging 环境）。注意进入 release 分支中就意味着待发布，应该是相对稳定的版本，不应该有太多 bug 修复。之后的部署、测试、bug 修复都是在 release 中进行。上线完成之后，release 分支也要合并到 develop 以及 master。release 分支更多的作用是，发布新版本时一些文件的变化，比如`package.json`版本号的改变，以及预发布时出现的小问题。另外，在多迭代开发的时候，开发并测试完一个迭代已经相对稳定之后但还未上线，可以从 develop checkout 出一个`release`。在 checkout 的那一刻起，develop 就可以接收新的迭代代码。

更多内容可以查看：[A successful Git branching model](https://jiongks.name/blog/a-successful-git-branching-model/)

### git tag

Git 使用两种标签：轻标签以及注解标签。轻标签只有名字，而注解标签可以有名字、注解以及签名。

一般情况下，发布标签是注解标签而轻标签是在本地暂时使用或一次性使用。

### git submodule

### git bisect
