---
layout: post
title:  "GitHub个人博客相关操作"
date:   2016-01-26 23:23:59
author: Jiang Biao
categories: Git
---
# 闲话几句

也许是早上起来太早，也许是一时兴起，也许是CU的博客界面实在有点ugly，重新在GitHub上整了这个博客，只为能记录下来工作中的点点滴滴，不至于辛苦劳累几年后，发现自己什么都没留下。

由于公司内不能用git，对git不熟，同时对web相关的东东也不熟，过程还是感觉比较折腾，但还是觉得很值得，自己又学到了不少东西。

时间不多，这里仅简单记录下在GitHub Pages中写博客的基本操作，以免后续忘了，也给大家做参考，如需了解相关的详细操作，还请google。


# Clone仓库
如果不想从头设计，只需要clone别人的仓库，比如：

	git clone https://github.com/HappySeeker/happyseeker.github.io.git

或者clone或下载开源的主题，jekyll官方提供了很多漂亮的主题，可以直接使用，也可以向其中贡献自己的主题:

[jekyll官方提供的主题](http://jekyllthemes.org/)

# 修改配置
修改原有仓库，形成自己的风格。需要主要的文件有：

- _config.yml --> 全局配置
- index.html --> 首页布局
- _posts目录 --> 存放文章
- _layouts目录 --> 页面布局


# 写文章
通常有markdown来写文章，语法很简单，注意文件名称的格式以及标题部分的格式，通常拷贝过来修改下即可。

# 检验文章
1. 使用jekyll来检验编写的文章是否能在浏览器中正常显示。 在文章所在的目录(_posts目录)中执行如下命令即可：

```
	jekyll serve --watch
```


2. 然后，打开浏览器，地址栏中输入localhost:4000即可看到自己的博客，检验新发布文章的效果。运行后，如果对文章有新的改动，不需要重新执行jekyll serve --watch命令，jekyll会自动监测修改，并重新build，你只需要刷新下浏览器网页就可以检验新的改动了。

# 提交仓库
写完文章后，需要将文章提交到GitHub仓库中，依次执行如下命令即可

	git st    --> 检查状态
	git add . --> 添加改动
	git commit -m 'your log'  --> 提交到本地仓库
	git push  --> 提交到远程(GitHub)仓库
