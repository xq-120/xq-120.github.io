---
title: github pages+hexo+next搭建博客
date: 2020-02-29 21:37:50
tags:
---



#### 主要步骤

* 安装环境
* 初始化项目
* 部署
* 配置站点信息
* 配置主题信息



下面是具体的步骤:

> 安装环境

安装node.js

安装hexo



> 初始化项目

使用 Hexo 的命令行创建一个项目.

`hexo init "name"`

调用 Hexo 的 generate 命令，将 Hexo 编译生成 HTML 代码.

`hexo generate`

利用 Hexo 提供的 server 命令让博客在本地运行起来.

`hexo server`



> 部署

安装一个支持 Git 的部署插件，名字叫做 hexo-deployer-git，有了它我们才可以顺利将其部署到 GitHub 上面，如果不安装的话，在执行部署命令时会报错误.

`npm install hexo-deployer-git --save`

部署

`hexo deploy`

部署成功之后,Hexo会将编译之后的静态页面内容推送到 GitHub 仓库的 master 分支.

需要注意的是里面是没有博客源码的.如果我们换了电脑就需要将以前的博客拷贝回来.这样显然很不方便,因此我们需要将博客源码也托管到 GitHub 上面.

**将博客源码托管到 GitHub 上面**

可以新建一个source分支,用于博客源码的管理.

```
git init
git checkout -b source
git add -A
git commit -m "init blog"
git remote add origin "https://github.com/xxx/xxx.github.io.git"
git push origin source
```



> 配置站点信息

站点简介修改



> 修改主题

选择一个主题使用.这里选择next主题.



> 主题配置

主题显示效果配置



> 写文章

新增文章

`hexo new "name"`

创建的文章会出现在 `source/_posts` 文件夹下，是 MarkDown 格式。

在文章开头通过如下格式添加必要信息：

```
---
title: 标题 # 自动创建，如 hello-world
date: 日期 # 自动创建，如 2020-03-01 13:30:15
tags: 
- 标签1
- 标签2
- 标签3
categories:
- 分类1
- 分类2
---
```



> 标签

增加标签页面

`hexo new page tags`



> 分类

增加分类页面

`hexo new page categories`



> 搜索页

安装搜索插件

`npm install hexo-generator-searchdb --save`

在项目的 _config.yml 里面添加搜索设置:

```
search:
  path: search.xml
  field: post
  format: html
  limit: 10000
```

在主题的 _config.yml 里面修改如下:

```
# Local search
# Dependencies: https://github.com/wzpan/hexo-generator-search
local_search:
  enable: true
  # If auto, trigger search by changing input.
  # If manual, trigger search by pressing enter key or search button.
  trigger: auto
  # Show top n results per article, show all results by setting to -1
  top_n_per_article: 5
  # Unescape html strings to the readable one.
  unescape: false
  # Preload the search data when the page loads.
  preload: false
```



> 404页面

设置自定义的404页面

其他还有很多可以配置的如评论系统,统计系统.



> 文章摘要

官方的推荐:在文章中使用 `<!-- more -->` 手动进行截断.建议使用 ``<!-- more -->``（即第一种方式），除了可以精确控制需要显示的摘录内容以外， 这种方式也可以让 Hexo 中的插件更好的识别。

不过以前的文章也不可能一一设置,可以使用下面的自动设置.

自动生成摘要的插件:

[Automatic excerpt generator for Hexo](https://github.com/chekun/hexo-excerpt)

安装:`npm install hexo-excerpt --save`



> RSS订阅

安装插件:

`npm install hexo-generator-feed --save`

然后在主题配置文件social一栏中新增一行`RSS: /atom.xml || rss`,就可以让rss显示在侧边栏了.注意不是`follow_me:`那里(害我搞了好久).

```yaml
# Social Links
# Usage: `Key: permalink || icon`
# Key is the link label showing to end users.
# Value before `||` delimiter is the target permalink, value after `||` delimiter is the name of Font Awesome icon.
social:
  GitHub: https://github.com/xq-120 || github
  #E-Mail: mailto:yourname@gmail.com || envelope
  #Weibo: https://weibo.com/yourname || weibo
  #Google: https://plus.google.com/yourname || google
  #Twitter: https://twitter.com/yourname || twitter
  #FB Page: https://www.facebook.com/yourname || facebook
  StackOverflow: https://stackoverflow.com/yourname || stack-overflow
  #YouTube: https://youtube.com/yourname || youtube
  #Instagram: https://instagram.com/yourname || instagram
  #Skype: skype:yourname?call|chat || skype
  RSS: /atom.xml || rss
```



#### 参考

[[利用 GitHub + Hexo + Next 从零搭建一个博客](https://cuiqingcai.com/7625.html)](https://cuiqingcai.com/7625.html)

[我的个人博客之旅：从jekyll到hexo](https://blog.csdn.net/u011475210/article/details/79023429)

[next配置官方说明](https://theme-next.iissnan.com/)

[next作者博客](https://notes.iissnan.com/)

[Hexo-NexT 主题个性优化](https://guanqr.com/tech/website/hexo-theme-next-customization/):这个很详细.