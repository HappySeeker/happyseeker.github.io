---
layout: post
title:  "Howto通过git向邮件列表中提交开源补丁"
date:   2017-07-06 09:43:32
author: JiangBiao
categories: Git
---

#  前言

社区贡献能体现公司价值，更能体现个人价值，目前很多大型的开源社区都通过邮件列表来提交补丁，典型的如Linux Kernl、Qemu、GNU之类的(也有其它一些社区更规范、直接使用git&gerrit来操作，本文暂不关注)，具体操作方式通常为：

1. 通过邮件客户端向指定的邮件列表中发补丁，并抄送相关的maintainer。
2. 相关模块的maintainer或相关人员进行code review。
3. 完成review(期间可能涉及反复讨论和修改)后，即由maintainer合入相应分支。

但是，在实际操作的过程中，往往会遇到补丁格式的问题(相信大部分初涉社区的TX都遇到过)，常常会因为补丁格式不对，受到其它人的抱怨或白眼。

# 原因

导致补丁格式问题的原因有很多，从补丁制作到补丁最终发送到邮件列表中，多个环境都可能导致格式问题：

1. 补丁制作。补丁制作的方式有多种，很多人喜欢直接diff命令，其它喜欢用git，甚至可能有其它做法，但是对于git管理的源代码来说，最标准的方式还是git，使用其它方式制作的补丁都有存在问题的可能性。

2. 拷贝粘贴。通常，如果通过邮件客户端发送补丁的化，需要将补丁从文本文档(或vim啥的)中拷贝到邮件内容中，拷贝的过程中，即可能发送格式变化，一些OS会在粘贴板中做一些处理。

3. 邮件客户端。很多邮件客户端都会对邮件内容进行格式化，进行格式调整，导致补丁格式错乱，比如之前用过的Notes和现在的Zmail，需要使用支持纯文本格式的客户端，防止做格式转换，但即便是纯文本格式，还是可能会做格式调整，依赖不可靠。

4. 邮件服务器。邮件从客户端发送到服务器的时候，邮件服务器还有可能根据内容做格式转换，虽然可能性比较小。

所以，要想把补丁原封不动的发到邮件列表中，并在格式上满足社区git合入需求，还是很不容易的。

# 标准的方式

那社区中的其它人是如何操作的呢？他们每天提交那么多补丁。

其实，社区中有标准的操作方式，完全使用git就能搞定。具体步骤如下：

## clone官方git仓库至本地

比如，对于内核，执行如下命令克隆：

	git clone https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git

当然，也可以使用其它命令，比如git fetch，该命令还有一个好处，能续传。

## 修改代码并提交

具体来说，有两种操作方式：

1. 创建新分支后修改。
2. 当前主分支修改。

通常建议创建一个分支，具体操作示例如下：

1. 创建分支

>[root@localhost git_kernel]# git branch dev 
	
2. 切换分支

>[root@localhost git_kernel]# git checkout dev
>切换到分支 'dev'

	
3. 修改代码，比如在MAINTAINERS文件第1行插入：
 
>####test for commit patch by git

>[root@localhost git_kernel]# git status
>位于分支 dev
>尚未暂存以备提交的变更：
>  （使用 "git add <文件>..." 更新要提交的内容）
>  （使用 "git checkout -- <文件>..." 丢弃工作区的改动）
>
>	修改：     MAINTAINERS
>
>修改尚未加入提交（使用 "git add" 和/或 "git commit -a"）

4. 将修改提交到本地（这步很关键）

>[root@localhost git_kernel]# git add MAINTAINERS 
>[root@localhost git_kernel]# git commit

在执行git commit后会出现一个VI格式的编辑器，在其中输入此次修改的说明，注意，这里的说明非常关键，输入的内容将直接应用到补丁中，所以，需要仔细编写，而且要注意格式，示例如：


	
	# 请为您的变更输入提交说明。以 '#' 开始的行将被忽略，而一个空的提交
	# 说明将会终止提交。
	# 位于分支 dev
	# 要提交的变更：
	#       修改：     MAINTAINERS
	#
	
格式描述需要按如下规则操作：
	- 第一行输入整体描述，该行直接对应于后续发送补丁邮件的标题，和git log中的标题。
	- 第一行输入后，空一行，然后在后面输入补丁的具体描述。
	- 补丁描述后，再空一行，然后在后面输入签名等信息(Signed-off-by:/Reviewed-by:等)，签名信息也可以在配置git后，通过git commit -s自动插入，感兴趣同学可以自己试试。
	- 每一行都不能超过70个字符，其实只需要对准下面#开始的注释的第一行长度即可，超过这个长度之前的最后一个单词应该提前换行。
	
示例如下：

	Add one line in file MAINTAINERS (This is he subject of email and  
	git log head)  
	 
	Add one line in file MAINTAINERS just for test.(You should add 
	your detailed description of your patch here)  
	 
	Signed-off-by: Jiang Biao <jiang.biao@zte.com.cn>  
	Reviewed-by: Another One <another@zte.com.cn>  
	# 请为您的变更输入提交说明。以 '#' 开始的行将被忽略，而一个空的提交
	# 说明将会终止提交。
	# 位于分支 dev
	# 要提交的变更：
	#       修改：     MAINTAINERS
                      
编辑完成后，按Vi的方式，输入:wq保存，即退出

## 使用git命令制作补丁

使用git自带的命令制作补丁，能最大程度保证补丁的格式，命令为：

	git format-patch
	
示例如：

>[root@localhost git_kernel]# git format-patch -M master
>0001-Add-one-line-in-file-MAINTAINERS-This-is-he-subject-.patch

意思上将当前分支(dev)上的所有commit与master分支做对比，制作补丁，如果commit了多次，会一起制作出多个补丁。补丁自动以commit中输入的第一行命名

看看补丁示例内容：

>[root@localhost git_kernel]# cat 0001-Add-one-line-in-file-MAINTAINERS-This-is-he-subject-.patch 

	From 33c9985a949809b3b8e5071c743bbf31201b1c2f Mon Sep 17 00:00:00 2001
	From: Jiang Biao <jiang.biao2@zte.com.cn>
	Date: Wed, 26 Jul 2017 16:12:56 +0800
	Subject: [PATCH] Add one line in file MAINTAINERS (This is he subject of email
	 and git log head)
	
	Add one line in file MAINTAINERS just for test.(You should add
	your detailed description of your patch here)
	
	Signed-off-by: Jiang Biao <jiang.biao@zte.com.cn>
	Reviewed-by: Another One <another@zte.com.cn>
	---
	 MAINTAINERS | 2 +-
	  1 file changed, 1 insertion(+), 1 deletion(-)
	
	diff --git a/MAINTAINERS b/MAINTAINERS
	index 767e9d2..bc8821d 100644
	--- a/MAINTAINERS
	+++ b/MAINTAINERS
	@@ -1,4 +1,4 @@
	-
	+####test for commit patch by git
	 
	 	List of maintainers and how to submit kernel changes
	 
	-- 
	2.7.4


这个补丁中包含了邮件需要的所有内容，包括ID，From、Date、主题、描述和补丁的具体内容，非常标准，后续直接将这个文件发送出去即可。

## 使用git发送补丁

git自带了邮件客户端功能，可以直接通过git发送补丁，如此，可以避免其它邮件客户端导致的各种格式问题。相应命令为：
	
	git send-email
	
但是，在使用这个功能之前，需要配置邮件服务器(不然谁来执行发送操作呢)，本质为配置smtp服务器和端口，分两种情况：

1. 在家里。家里的环境可以直接连接公网，可以将直接设置开放的大众邮件服务器，比如163、126、hotmail、gmail，具体配置方法这里不描述了，感兴趣同学可以自己google一下，应该不复杂。

2. 在公司。对应我们这样公司环境来说，所有端口都受限，无法直接使用公网的邮件服务器，只能使用公司自己的邮件服务器来发邮件，此时需要在IT网站上申请一个SMTP帐号，审批后即可配置使用公司自己的邮件服务器，配置示例如：

>[sendemail]
>        smtpencryption = tls
>        smtpuser = JiangBiao
>        smtpserverport = 25
>        smtpserver = "xxsmtpxx.xxx.com.cn"

当然帐号申请会有限制，不建议大家都去申请，如果是临时使用，可以大家共用帐号和服务器。如果觉得不方便，也可以将补丁发给我代为提交(签名不变)。

配置完成后，就可以使用git发送邮件了，示例如下：

>git send-email --compose --no-chain-reply-to --suppress-from --to linux-kernel@vger.kernel.org --cc maintainer@xxx.com --cc jiang.biao2@zte.com.cn 0001-Add-one-line-in-file-MAINTAINERS-This-is-he-subject-.patch

发送前会提示你确认补丁内容、邮件内容，还是Vi格式，确认后输入:wq继续；如果有误，则直接:q!退出修改后重新发送。

# 效果

补丁从制作到发送过程如此，基本完全通过git搞定，这也是社区的标准方式，如此操作，发送的补丁基本不可能出现格式问题了。



