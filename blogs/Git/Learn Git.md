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

