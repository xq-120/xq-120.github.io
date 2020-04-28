参考：

[Git官方使用教程](https://git-scm.com/book/zh/v2)

[Git权威指南](http://www.worldhello.net/gotgit/)

[这才是真正的Git——Git内部原理揭秘！](https://zhuanlan.zhihu.com/p/96631135)

查看git的安装路径

`where git `

```
$ where git
/usr/local/bin/git
/usr/bin/git
```

可以看到安装了两个git软件。`/usr/local/bin/git`该目录下的其实是个快捷键，真实路径为`/usr/local/git/bin/git`。

`which git`，查看实际使用的是哪个git。

```
$ which git
/usr/local/bin/git
```

## 初次运行 Git 前的配置

既然已经在系统上安装了 Git，你会想要做几件事来定制你的 Git 环境。 每台计算机上只需要配置一次，程序升级时会保留配置信息。 你可以在任何时候再次通过运行命令来修改它们。

Git 自带一个 `git config` 的工具来帮助设置控制 Git 外观和行为的配置变量。 这些变量存储在三个不同的位置：

1. `/etc/gitconfig` 文件: 包含系统上每一个用户及他们仓库的通用配置。 如果在执行 `git config` 时带上 `--system` 选项，那么它就会读写该文件中的配置变量。 （由于它是系统配置文件，因此你需要管理员或超级用户权限来修改它。）
2. `~/.gitconfig` 或 `~/.config/git/config` 文件：只针对当前用户。 你可以传递 `--global` 选项让 Git 读写此文件，这会对你系统上 **所有** 的仓库生效。
3. 当前使用仓库的 Git 目录中的 `config` 文件（即 `.git/config`）：针对该仓库。 你可以传递 `--local` 选项让 Git 强制读写此文件，虽然默认情况下用的就是它。。 （当然，你需要进入某个 Git 仓库中才能让该选项生效。）

每一个级别会覆盖上一级别的配置，所以 `.git/config` 的配置变量会覆盖 `/etc/gitconfig` 中的配置变量。

你可以通过以下命令查看所有的配置以及它们所在的文件：

```console
$ git config --list --show-origin
```

如下：

```
...
file:/usr/local/git/etc/gitconfig	include.path=~/.gitcinclude
file:/usr/local/git/etc/gitconfig	include.path=.githubconfig
file:/usr/local/git/etc/gitconfig	include.path=.gitcredential
file:/usr/local/git/etc/gitconfig	diff.exif.textconv=exif
file:/usr/local/git/etc/gitconfig	credential.helper=osxkeychain
...

file:/Users/xuequan/.gitconfig	core.excludesfile=/Users/xuequan/.gitignore_global
file:/Users/xuequan/.gitconfig	difftool.sourcetree.cmd=opendiff "$LOCAL" "$REMOTE"
file:/Users/xuequan/.gitconfig	difftool.sourcetree.path=
file:/Users/xuequan/.gitconfig	mergetool.sourcetree.cmd=/Applications/SourceTree.app/Contents/Resources/opendiff-w.sh "$LOCAL" "$REMOTE" -ancestor "$BASE" -merge "$MERGED"
file:/Users/xuequan/.gitconfig	mergetool.sourcetree.trustexitcode=true
file:/Users/xuequan/.gitconfig	commit.template=/Users/xuequan/.stCommitMsg
file:/Users/xuequan/.gitconfig	http.postbuffer=524288000
```

可以看到macOS上，git系统配置文件在git的安装目录`/usr/local/git`下的`/etc/gitconfig`。git全局配置文件在`~/.gitconfig`下。

如果不需要这么详细的日志，可以使用`git config --list`：

```
$ git config --list
core.excludesfile=~/.gitignore
core.legacyheaders=false
core.quotepath=false
core.pager=less
mergetool.keepbackup=true
push.default=simple
color.ui=auto
color.interactive=auto
repack.usedeltabaseoffset=true
alias.s=status
alias.a=!git add . && git status
alias.au=!git add -u . && git status
alias.aa=!git add . && git add -u . && git status
alias.c=commit
alias.cm=commit -m
alias.ca=commit --amend
...
```

**更改配置**

> 使用命令

安装完 Git 之后，要做的第一件事就是设置你的用户名和邮件地址。 这一点很重要，因为每一个 Git 提交都会使用这些信息，它们会写入到你的每一次提交中，不可更改：

```console
$ git config --global user.name "John Doe"
$ git config --global user.email johndoe@example.com
```

再次强调，如果使用了 `--global` 选项，那么该命令只需要运行一次，因为之后无论你在该系统上做任何事情， Git 都会使用那些信息。 当你想针对特定项目使用不同的用户名称与邮件地址时，可以在那个项目目录下运行没有 `--global` 选项的命令来配置。

> 直接打开配置文件修改

比如进入全局配置文件目录，然后执行`vim .gitconfig`，直接编辑。

### Git三种状态

工作区，暂存区，Git目录区。

[1.3 起步 - Git 是什么？](https://git-scm.com/book/zh/v2/起步-Git-是什么？)

新建一个文件`touch README`，再使用`git status`查看：

```
$ git status
位于分支 master
您的分支与上游分支 'origin/master' 一致。

未跟踪的文件:
  （使用 "git add <文件>..." 以包含要提交的内容）
	README

提交为空，但是存在尚未跟踪的文件（使用 "git add" 建立跟踪）
```

README处于未跟踪状态。

使用命令 `git add` 开始跟踪一个文件。 所以，要跟踪 `README` 文件，运行：

```console
$ git add README
```

此时再运行 `git status` 命令，会看到 `README` 文件已被跟踪，并处于暂存状态：

```
$ git status
位于分支 master
您的分支与上游分支 'origin/master' 一致。

要提交的变更：
  （使用 "git restore --staged <文件>..." 以取消暂存）
	新文件：   README
```

此时README就是已暂存状态。根据提示使用`git restore --staged <文件>`可以取消暂存

```
git restore --staged README
```

再次使用git status：

```
$ git status
位于分支 master
您的分支与上游分支 'origin/master' 一致。

未跟踪的文件:
  （使用 "git add <文件>..." 以包含要提交的内容）
	README

提交为空，但是存在尚未跟踪的文件（使用 "git add" 建立跟踪）
```

这就是**取消跟踪暂存区的文件**操作。

### 移除文件

要从 Git 中移除某个文件，就必须要从已跟踪文件清单中移除（确切地说，是从暂存区域移除），然后提交。可以用 `git rm` 命令完成此项工作，该命令会顺带将文件从工作目录中删除且不会出现在回收站里（类似于rm命令）。

eg:

```
git rm CONTRIBUTING.md
git commit -m "xxx"
```

可以看到上述过程并没有使用到常规的rm命令。

如果只是简单地从工作目录中手工删除文件，运行 `git status` 时就会在 “Changes not staged for commit” 部分（也就是 *未暂存清单*）看到：

```console
$ rm PROJECTS.md
$ git status
On branch master
Your branch is up-to-date with 'origin/master'.
Changes not staged for commit:
  (use "git add/rm <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

        deleted:    PROJECTS.md

no changes added to commit (use "git add" and/or "git commit -a")
```

此时还是需要再运行 `git rm` 记录此次移除文件的操作：

```console
$ git rm PROJECTS.md
rm 'PROJECTS.md'
$ git status
On branch master
Your branch is up-to-date with 'origin/master'.
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

    deleted:    PROJECTS.md
```

然后提交。

> 想把文件从 Git 仓库中删除（亦即从暂存区域移除），但仍然希望保留在当前工作目录中。 换句话说，你想让文件保留在磁盘，但是并不想让 Git 继续跟踪。

为达到这一目的，使用 `--cached` 选项：

```console
$ git rm --cached README
```

`git rm` 命令后面可以列出文件或者目录的名字，也可以使用 `glob` 模式。比如：

```console
$ git rm log/\*.log
```

注意到星号 `*` 之前的反斜杠 `\`， 因为 Git 有它自己的文件模式扩展匹配方式，所以我们不用 shell 来帮忙展开。 此命令删除 `log/` 目录下扩展名为 `.log` 的所有文件。 类似的比如：

```console
$ git rm \*~
```

该命令会删除所有名字以 `~` 结尾的文件。

## 撤消操作

> git commit --amend

有时候我们提交完了才发现漏掉了几个文件没有添加，或者提交信息写错了。 此时，可以运行带有 `--amend` 选项的提交命令来重新提交：

```console
$ git commit --amend
```

**这个命令会将暂存区中的文件提交**。 如果自上次提交以来你还未做任何修改（例如，在上次提交后马上执行了此命令）， 那么快照会保持不变，而你所修改的只是提交信息。

eg：

`git commit --amend -m " 修改提交信息"`

前

```
commit 2f47e7f9e72bca7514a8de4d08f5cbdd928b9993
Author: xuequan <quan.xue@zhenai.com>
Date:   Mon Apr 27 11:32:45 2020 +0800

    amend修改README.md

diff --git a/README.md b/README.md
index 51480af..f5c5178 100644
--- a/README.md
+++ b/README.md
@@ -2,3 +2,4 @@

 hello,world!
 111
+222

commit c4f103d17fa3711a33ce24d8664d3c6473ae280a
Author: xuequan <quan.xue@zhenai.com>
Date:   Mon Apr 27 11:31:21 2020 +0800

    重新添加README.md

diff --git a/README.md b/README.md
new file mode 100644
index 0000000..51480af
--- /dev/null
+++ b/README.md
@@ -0,0 +1,4 @@
+# git_demo
+
+hello,world!
+111

```

后：可以看到提交评论已经从”amend修改README.md“变成”修改提交信息“了。

```
commit 3041119e5670c9daa1f23d6cb2018ef2b9ce56ff
Author: xuequan <quan.xue@zhenai.com>
Date:   Mon Apr 27 11:32:45 2020 +0800

    修改提交信息

diff --git a/README.md b/README.md
index 51480af..f5c5178 100644
--- a/README.md
+++ b/README.md
@@ -2,3 +2,4 @@

 hello,world!
 111
+222

commit c4f103d17fa3711a33ce24d8664d3c6473ae280a
Author: xuequan <quan.xue@zhenai.com>
Date:   Mon Apr 27 11:31:21 2020 +0800

    重新添加README.md

diff --git a/README.md b/README.md
new file mode 100644
index 0000000..51480af
--- /dev/null
+++ b/README.md
@@ -0,0 +1,4 @@
+# git_demo
+
+hello,world!
+111

```

如果我们提交完了才发现漏掉了几个文件没有添加或者又想修改点啥：

```
修改了点东西

git add README.md

git commit --amend -m " 修改上一次提交：又修改了一点README.md"

```

日志如下：可以看到我们又修改的东西已经在这一次提交上了，并且上一次的提交已经没有了。

```
commit 230a0a79f1bbc72053dc4824f66facffb93d7a84
Author: xuequan <quan.xue@zhenai.com>
Date:   Mon Apr 27 11:32:45 2020 +0800

    修改上一次提交：又修改了一点README.md

diff --git a/README.md b/README.md
index 51480af..60b5084 100644
--- a/README.md
+++ b/README.md
@@ -2,3 +2,5 @@

 hello,world!
 111
+222
+333

commit c4f103d17fa3711a33ce24d8664d3c6473ae280a
Author: xuequan <quan.xue@zhenai.com>
Date:   Mon Apr 27 11:31:21 2020 +0800

    重新添加README.md

diff --git a/README.md b/README.md
new file mode 100644
index 0000000..51480af
--- /dev/null
+++ b/README.md
@@ -0,0 +1,4 @@
+# git_demo
+
+hello,world!
+111

```

### 取消暂存的文件

`git reset HEAD <File>`

例如，你已经修改了两个文件并且想要将它们作为两次独立的修改提交， 但是却意外地输入 `git add *` 暂存了它们两个。如何只取消暂存两个中的一个呢？

```
$ git status
On branch master
Your branch is ahead of 'origin/master' by 1 commit.
  (use "git push" to publish your local commits)
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

	new file:   CONTRIBUTING.md
	modified:   README.md

```

执行`git reset HEAD CONTRIBUTING.md`

```
$ git status
On branch master
Your branch is ahead of 'origin/master' by 1 commit.
  (use "git push" to publish your local commits)
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

	modified:   README.md

Untracked files:
  (use "git add <file>..." to include in what will be committed)

	CONTRIBUTING.md

```

可以看到CONTRIBUTING.md已经不在暂存区了。

### 撤消对文件的修改

`git checkout -- <file>`

提交之后发现有些改的地方有问题想撤销上次的修改（即将它还原成上次提交时的样子），怎么办？

`git checkout -- CONTRIBUTING.md`

请务必记得 `git checkout -- ` 是一个危险的命令。 你对那个文件在本地的任何修改都会消失——Git 会用最近提交的版本覆盖掉它。 除非你确实清楚不想要对那个文件的本地修改了，否则请不要使用这个命令。

### 远程仓库

 与他人协作涉及管理远程仓库以及根据需要推送或拉取数据。 管理远程仓库包括了解**如何添加远程仓库**、**移除无效的远程仓库**、**管理不同的远程分支并定义它们是否被跟踪**等等。

#### 查看远程仓库

如果想查看你已经配置的远程仓库服务器，可以运行 `git remote` 命令。 它会列出你指定的每一个远程服务器的简写。 如果你已经克隆了自己的仓库，那么至少应该能看到 origin ——这是 Git 给你克隆的仓库服务器的默认名字：

```
$ git remote
origin
```

我这里只有git clone时默认的origin。如果一个本地仓库没有添加任何远程仓库，则该命令不会有任何反应。

可以指定选项 `-v`，会显示需要读写远程仓库使用的 Git 保存的简写与其对应的 URL。

```
$ git remote -v
origin	https://github.com/xq-120/git_demo.git (fetch)
origin	https://github.com/xq-120/git_demo.git (push)
```

查看某个远程仓库

如果想要查看某一个远程仓库的更多信息，可以使用 `git remote show ` 命令。 如果想以一个特定的缩写名运行这个命令，例如 `origin`，会得到像下面类似的信息：

```
$ git remote show origin
* remote origin
  Fetch URL: https://github.com/xq-120/git_demo.git
  Push  URL: https://github.com/xq-120/git_demo.git
  HEAD branch: master
  Remote branch:
    master tracked
  Local branch configured for 'git pull':
    master merges with remote master
  Local ref configured for 'git push':
    master pushes to master (up to date)
```

它同样会列出远程仓库的 URL 与跟踪分支的信息。它告诉你正处于远程仓库的 `master` 分支，并且如果运行 `git pull`， 就会抓取所有的远程引用，然后将远程 `master` 分支合并到本地 `master` 分支。 它也会列出拉取到的所有远程引用。

#### 添加远程仓库

运行 `git remote add <shortname> <url>  ` 添加一个新的远程 Git 仓库，同时指定一个方便使用的简写：

`git remote add pb https://github.com/paulboone/ticgit`

以后可以在命令行中使用字符串 `pb` 来代替整个 URL。

使用git clone:

```
$ git remote show origin
* remote origin
  Fetch URL: https://github.com/xq-120/git_demo.git
  Push  URL: https://github.com/xq-120/git_demo.git
  HEAD branch: master
  Remote branch:
    master tracked
  Local branch configured for 'git pull':
    master merges with remote master
  Local ref configured for 'git push':
    master pushes to master (up to date)
```

手动添加远程仓库：远程分支master没有被自动跟踪

```
$ git remote add genius https://github.com/xq-120/git_demo.git
$ git remote show genius
* remote genius
  Fetch URL: https://github.com/xq-120/git_demo.git
  Push  URL: https://github.com/xq-120/git_demo.git
  HEAD branch: master
  Remote branch:
    master new (next fetch will store in remotes/genius)
  Local ref configured for 'git push':
    master pushes to master (up to date)
```

此时运行git pull/push会报错：

```
$ git pull
fatal: No remote repository specified.  Please, specify either a URL or a
remote name from which new revisions should be fetched.
$ git push genius
fatal: The current branch master has no upstream branch.
To push the current branch and set the remote as upstream, use

    git push --set-upstream genius master
```

这时就必须指明远程仓库别名和分支`git pull genius master`。

每次指明远程仓库别名和分支必然不方便，可以设置git push和pull的默认分支：

```
git branch --set-upstream-to=genius/master master
```

这个时候就可以省略了。

#### 从远程仓库中抓取与拉取

git clone做了哪些事？

1.将远程仓库拷贝到本地

2.自动执行了`git remote add origin <url>  `，即自动为本地仓库添加刚才的远程仓库并默认以 “origin” 为简写。 

3.自动设置本地 master 分支跟踪克隆的远程仓库的 `master` 分支（或其它名字的默认分支）。

从远程仓库中获得数据，可以执行：

```console
$ git fetch <remote>
```

`git fetch origin` 会抓取克隆（或上一次抓取）后新推送的所有工作。 必须注意 `git fetch` 命令只会将数据下载到你的本地仓库——**它并不会自动合并或修改你当前的工作**。 当准备好时你必须手动将其合并入你的工作（使用git merge）。

如果你的当前分支设置了跟踪远程分支（阅读下一节和 [Git 分支](https://git-scm.com/book/zh/v2/ch00/ch03-git-branching) 了解更多信息）， 那么可以用 `git pull` 命令来自动抓取后合并该远程分支到当前分支。

可以看出git pull = git fetch + git merge

#### 推送到远程仓库

当你想分享你的项目时，必须将其推送到上游。 这个命令很简单：`git push  `。 当你想要将 `master` 分支推送到 `origin` 服务器时（再次说明，克隆时通常会自动帮你设置好那两个名字）， 那么运行这个命令就可以将你所做的备份到服务器：

```console
$ git push origin master
```

只有当你有所克隆服务器的写入权限，并且之前没有人推送过时，这条命令才能生效。 当你和其他人在同一时间克隆，他们先推送到上游然后你再推送到上游，你的推送就会毫无疑问地被拒绝。 你必须先抓取他们的工作并将其合并进你的工作后才能推送。

我们在平时只需要git push/pull，都是因为克隆时通常会自动帮你设置好那两个名字，所以可以省略的原因。

#### 远程仓库的重命名与移除

重命名

```console
git remote rename pb paul
```

移除

`git remote rm paul`

一旦你使用这种方式删除了一个远程仓库，那么所有和这个远程仓库相关的远程跟踪分支以及配置信息也会一起被删除。

### 疑惑	

1.git add与git stash

git add是将文件添加到暂存区，而git stash，stash（贮藏）是将工作区和暂存区的改动暂时保存到一个栈上，保存之后工作区就是上一次提交后的干净的工作区了。二者操作的物理文件是不一样的。

`.git/`目录如下：

```
total 56
-rw-r--r--   1 xuequan  staff    7  4 27 00:05 COMMIT_EDITMSG
-rw-r--r--   1 xuequan  staff   23  4 26 23:51 HEAD
-rw-r--r--   1 xuequan  staff   41  4 27 23:26 ORIG_HEAD
-rw-r--r--   1 xuequan  staff  308  4 26 23:51 config
-rw-r--r--   1 xuequan  staff   73  4 26 23:51 description
drwxr-xr-x  14 xuequan  staff  448  4 26 23:51 hooks
-rw-r--r--   1 xuequan  staff  217  4 27 23:26 index
drwxr-xr-x   3 xuequan  staff   96  4 26 23:51 info
drwxr-xr-x   4 xuequan  staff  128  4 26 23:51 logs
drwxr-xr-x  14 xuequan  staff  448  4 27 23:26 objects
-rw-r--r--   1 xuequan  staff  114  4 26 23:51 packed-refs
drwxr-xr-x   6 xuequan  staff  192  4 27 23:26 refs
```

index文件就是常说的暂存区。

执行git stash后，在`.git/refs/`目录下会出现一个`stash`文件：

```
$ ls -l
total 8
drwxr-xr-x  3 xuequan  staff  96  4 27 23:26 heads
drwxr-xr-x  3 xuequan  staff  96  4 26 23:51 remotes
-rw-r--r--  1 xuequan  staff  41  4 27 23:26 stash
drwxr-xr-x  2 xuequan  staff  64  4 26 23:51 tags
```

可见二者操作的确实不是同一个东西。git stash我之前一直理解的是暂存，进而想当然的认为是把工作区和暂存区的修改暂存到暂存区，然后就发现跟git add不就功能重叠了吗？然后就一直纠结搞不明白。现在终于明白了。