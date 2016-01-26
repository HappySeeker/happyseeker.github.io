---
layout: post
title:  "GitHub博客中文索引和中文字体问题"
date:   2016-01-26 23:45:59
author: Jiang Biao
categories: Git
---
# 索引问题

GitHub Pages+Jekyll的博客中，首页通常使用paginate进行分页，首页中每篇文章都希望只显示索引，否则会将整个文章内容都显示到首页上，显得凌乱不堪，而通常使用truncatewords来实现索引并限制索引的长度，如:

	post.content | strip_html | truncatewords: 50
			

如此的问题在于truncatewords只支持英文单词，即限制英文单词在索引中的数量，而中文不能支持。

# 解决
truncatewords支持英文的本质在于以**空格**为检测标记，所以，如果不想麻烦地去针对中文实现单独的truncatewords接口的话，完全可以利用**空格+truncatewords**来限制中文索引，只需要将truncatewords数值改小，比如5:

	post.content | strip_html | truncatewords: 5
	

然后根据自己的习惯，在写文章时，在句号后面多加一个空格，或者在所有标点符号后面都多加一个空格，即可达到目的。

如果是在句号后面多加一个空格，那么truncatewords:5即表示在索引中显示5句话。

# 中文字体问题

从jekyll官方获取的模板中所有的字体都是针对英文设置的，通常使用google font，设置的英文字体很漂亮，但google font不支持中文字体，所以如果博客是中文的问题，中文就是默认的比较难看的字体了。

# 解决
需要自己手工设置字体，比如个人比较偏爱的黑体，通常是在相应的css文件中设置，比如我的博客中设置的位置为：

	_sass\base\_variables.scss

原有字体集为：

	$base-font-family: "Open Sans", "Helvetica Neue", "Helvetica", "Roboto", "Arial", sans-serif; // $helvetica;
	$heading-font-family: "Roboto Slab", "Helvetica Neue", "Helvetica", "Arial", sans-serif; // $base-font-family;


修改后字体集为：

	$base-font-family: "Microsoft YaHei", "MicrosoftJhengHei", STHeiti, MingLiu, "Open Sans", "Helvetica Neue", "Helvetica", "Roboto", "Arial", sans-serif; // $helvetica;
	$heading-font-family: "Microsoft YaHei", "MicrosoftJhengHei", "Roboto Slab", "Helvetica Neue", "Helvetica", "Arial", sans-serif; // $base-font-family;