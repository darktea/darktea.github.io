---
layout: post
title: 利用 Gitbug Pages 写 Blog
category: notes
---

利用 [Github] Pages 写 Blog? 网上关于这个主题的介绍已经太多了, 这里我只是重复造轮子, 记录一下 Notes 而已.

看懂这篇 Blog 有一个前提: **你已经会使用 Github**

1. 到 Github 注册一个新帐号
1. 新创建一个 repository. 而这个 repository 的名字必须是 XXXXX.github.io, 其中 XXXXX 是上一步注册的 Github 的帐号的名字
1. 准备测试文件: 上传一个 index.html 文件到这个 repository 的根目录 (index.html 的内容随便写一点测试内容就行了)
1. 测试效果: 把 index.html 用 git push 命令上传到 repository 中. 等大概10分钟, 访问 http://XXXXX.github.io 就可以看到 index.html 了 (XXXXX 是前面创建好的 Github 的帐号名字)

上面的步骤只是试通 Github Pages 的最基本的功能: 显示最简单的 html 文件.

* 如果要让 Blog 有更美观的界面, 更强大的功能 (加评论什么的), 可以直接利用 Jekyll, Github Pages 内置了对 [Jekyll] 的支持.
* 我们只需要按 Jekyll 的模版格式上传我们的自己的定义的模版文件, Github Pages 就会自动依据这些模版文件帮你生成 Blog 内容.
* 但这里有一个问题, 作为小白的我, 不熟悉 Jekyll 的模版格式, 我怎么创建自己的定义的模版文件呢? 这里, 我利用了这个项目: https://github.com/mytharcher/SimpleGray

下面就介绍一下怎么利用 https://github.com/mytharcher/SimpleGray 来创建自己的 Blog 的模版文件

1. 利用 git clone 命令把 https://github.com/mytharcher/SimpleGray 整个工程下载下来
1. 删除 SimpleGray 下面的 .git 目录
1. 把 SimpleGray 目录下面的所有文件和目录, 利用 git push 提交到自己的 repository 里面 (XXXXX.github.io)
1. 只需要修改一个文件 XXXXX.github.io\_config.yml (具体怎么改, 参考 https://github.com/darktea/darktea.github.io/blob/master/_config.yml)
1. 然后把修改用 git push 到 Github 上面去就行了

最后, 重新刷新一下 Blog 看看修改的效果