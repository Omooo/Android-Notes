---
Learn Git
---

#### 技巧

```java
// 查看最近的线性提交历史，省着看一大坨提交历史，一般情况下我们只需要看 git message 即可
git log --oneline
输出：
// ...    
570f4a6 add git ignore file
578f96b Init commit

// 其实还可以指定查看最近的几个：
git log -n5 --oneline
```

```java
// 比较暂存区和 HEAD 之间的区别
git diff --cached
// 比较工作区和暂存区之间的区别
git diff
```

```java
git reset 有三个参数
--soft 这个只是把 HEAD 指向的 commit 恢复到你指定的 commit，暂存区 工作区不变
--hard 这个是 把 HEAD， 暂存区， 工作区 都修改为 你指定的 commit 的时候的文件状态
--mixed 这个是不加时候的默认参数，把 HEAD，暂存区 修改为 你指定的 commit 的时候的文件状态，工作区保持不变
```

```java
// git checkout 不仅可以切分支，还可以使工作区的内容保持和暂存区一致
// 恢复 readme.md 文件和暂存区一致
git checkout -- readme.md
// 小结：
// 已暂存区为中心
// 暂存区与HEAD比较：git diff --cached
// 暂存区与工作区比较: git diff
// 暂存区恢复成HEAD : git reset HEAD
// 暂存区覆盖工作区修改：git checkout
```



#### 情景模式

情景一：当多个仓库使用的用户名和 email 不一致时

比如你自己 GitHub 的仓库和公司的私有库，两个仓库的提交者的 name 和 email 是不一致的，这时我们一般会设置一个全局的 name 和 email 用于提交公司代码：

```java
git config --global user.name "myname"
git config --global user.email "myname@company.com"
```

--global 参数对当前用户所有仓库有效。

但是如果我们在自己的 GitHub 仓库提交时，还需要改 user.name 和 user.email。

其实有一种简单的办法，即一个 --local 参数，可以配置只对某个仓库。

比如在我的 Android-Note 目录下配置：

```jade
git config --local user.name "Omooo"
git config --local user.email "869759698@qq.com"
```

这样，我每次在这个仓库提交时，就会使用这个本地的 name 和 email。



情景二：修改上一次的提交

我提交了一个 commit，但并没有 push 远程仓库，这时候我发现 commit msg 里写的版本号出错了，这时候怎么修改上一次提交呢？其实很简单：

```java
git commit --amend
```

--amend 参数表示是修改上一次提交，而不是产生一个新的提交。

同理，这也可以用于多次修改一个提交。比如我们再往远程 push 时，我们希望只产生一个提交，这时候在本地开发时，可以多次 commit --amend 而不是多次 commit，这样就永远只会有一个提交了。



情景三：合并多个提交为一个提交

在我们自己的开发分支上可能有多个提交，一般在 QA 测试完之后，我们需要把本地分钟的提交合并到 master，一般不推荐直接 merge，而是切到 master，把我们自己的 commit pick 到 master 上。这时候就需要将我们的开发分支的多个提交压成一个提交，比如将最近的的三个提交压成一个提交：

```java
git rebase -i HEAD~3
```

然后就会有以下输出：

```java
pick 0f8d0e5 first commit
pick 5beec24 second commit
pick ab08235 third commit

# Rebase da9da1b..a28b190 onto 0f8d0e5 (6 commands)
#
# Commands:
# p, pick <commit> = use commit
# r, reword <commit> = use commit, but edit the commit message
# e, edit <commit> = use commit, but stop for amending
# s, squash <commit> = use commit, but meld into previous commit
# f, fixup <commit> = like "squash", but discard this commit's log message

```

这时候输入 I 进入编辑模式，改成以下：

```
pick 0f8d0e5 first commit
s 5beec24 second commit
s ab08235 third commit

# Rebase da9da1b..a28b190 onto 0f8d0e5 (6 commands)
#
# Commands:
# p, pick <commit> = use commit
# r, reword <commit> = use commit, but edit the commit message
# e, edit <commit> = use commit, but stop for amending
# s, squash <commit> = use commit, but meld into previous commit
# f, fixup <commit> = like "squash", but discard this commit's log message

```

也就是把 second 和 third commit 合并到 first commit 上。下面还有一些参数可以使用，比如说 r 就是保留提交，但是需要编辑 commit message。最后 esc :wq 保存就行了。

