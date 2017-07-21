---
layout: post
title:  "Git笔记：版本回退"
date:   2017-07-06 09:43:32
author: JiangBiao
categories: Git
---

#  场景

使用git合版本时，遇到这样的场景：

在合入了多个补丁后，发现之前合入的莫个补丁有问题，如：

		O->P1->P2->P3
  
  O表示初始代码，P1-P3分别对应3次补丁合入。合入P3后，发现P1补丁有问题，此时怎么办？
  
# 解决

## Easyest way
  
最简单的处理方法，当然就是重新提及一个补丁来修复P1补丁中的问题。

此时，只需要在代码中进行修改，然后git add / commit /push 即可

## Not so easy

更复杂的情况，当P1补丁非常复杂，修复成本很高，且P2和P3补丁严重依赖于P1补丁时，比如：P1补丁是将源代码中的某tar解压，然后删掉tar包，而在P3合入之后，你突然发现P1中解压的tar包版本有问题，需要使用另一个版本的tar包，此时直接提交新补丁来修复P1补丁，会非常麻烦，而且会对P2和P3补丁带来影响(P2/P3补丁可能需要重新打)

那该怎么做？

分两种情况：

### 未Push

在还未Push之前，所有commit在本地，此时可以：

1. 使用git format-patch -1 <commit> 分别针对P2合P3制作两个补丁。目的是避免重新修改代码、提交。 
2. 使用git reset --hard回退至O状态。
3. 重新提交P1补丁(新的tar包解压结果)
4. 使用git am <补丁名> 重新打上P2和P3补丁(步骤1中已经生成)

### 已Push

Push后，所有commit已经同步到remote，此时如果需要回退，则只能使用git revert命令，revert与reset不同，reset只是移动HEAD，而revert则为之前commit的补丁的逆操作，本质上是针对之前的patch制作一个新的patch来去除原patch所做的修改。

操作步骤与未Push情况类似：

1. 使用git format-patch -1 <commit> 分别针对P2合P3制作两个补丁。目的是避免重新修改代码、提交。 
2. 使用git revert回退至O状态。
3. 重新提交P1补丁(新的tar包解压结果)
4. 使用git am <补丁名> 重新打上P2和P3补丁(步骤1中已经生成)
5. git push(有gerrit情况下，使用git review)