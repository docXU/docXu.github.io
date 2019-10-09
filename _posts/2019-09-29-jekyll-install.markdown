---
layout:     post
title:      "Welcome to Matt's Blog"
subtitle:   " \"Hello World, Hello Blog\""
date:       2019-09-29 19:55:00
author:     "MattX"
header-img: "img/post-bg-2015.jpg"
tags:

    - 生活

---

> 我开始了。

这就是阿伟的个人博客，开篇记录下搭建步骤。

[跳过废话，直接看搭建步骤](#jekyll和github)

## 前言

这个主题是前端大佬Hux定制开源，记得还是某日从小马哥的直播录像看到他的博客，被这个博客主题干净清晰的风格闪瞎了狗眼，于是顺着网线一路找到了[地址](https://github.com/Huxpro/huxpro.github.io)

2019年初的KPI里我写了今年要开个博客，把平时工作和生活中所见所得所想找个地方记录，到今天终于想起来了，平时碰到需要记录的时刻大多是记在印象笔记里要么打开手机便签记忆几个关键字然后过了一会就断了思绪。

这可不行啊，这种稍纵即逝的思维习惯很危险啊，奉劝各位读者朋友们，如果你像我一样，经常在思考某件事的时候能有一些想法，最好还是找个方式记下来，在写作的过程中，也能帮助我们加深记忆，完事有些水友还会评论分享，自然也能从留言里听取一些建议。

---

## jekyll和github

jekyll是一个静态网站生成器（可以模板式的将文本转换为html网页）

我们都知道github提供了个人项目的页面展示功能（github pages），而且还提供的无限空间和个人域名（free~）

首先复制一份代码，建议直接Fork [Hux的项目](https://github.com/Huxpro/huxpro.github.io) （带有很多博文，可以学一下md的语法嘿嘿），不要去做Boilerplate的（已经不维护了，有很多bug）

然后安装jekyll，现在是ruby管理的，所以我们搭建ruby环境，在[本地运行jekyll](https://blog.csdn.net/mouday/article/details/79300135)实时预览页面生成后的效果

> 热部署 jekyll server --watch 

最后根据github的[规范](https://www.jianshu.com/p/b6dfc7c886a9)创建展示项目

在_posts下发布md博客，确认无误后git flow就完事了。


## Q&A

1. 推送posts后没看到博文？

    因为jekyll需要重新编译生成静态页面+github展示页本身需要资源分配，所以无法立即看到，稍等片刻后刷新。（等待时间和文章数量有关）

---

