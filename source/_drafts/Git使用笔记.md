参考：

[Git官方使用教程](https://git-scm.com/book/zh/v2)

[Git权威指南](http://www.worldhello.net/gotgit/)



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

