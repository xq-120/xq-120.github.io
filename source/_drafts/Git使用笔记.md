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

