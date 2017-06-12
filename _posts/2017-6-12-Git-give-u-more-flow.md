---
layout: post
title: "git让你拥有更多的流之利用gitlab进行代码review"
date: 2017-6-12 12:00:00
categories: [Git]
---

Let's 直接进入正题。。。

首先简单说一下为什么要做代码的review。

1. 能够让review的人更加的了解你开发的代码
2. 能够让review的人发现一些自身未发现的bug
3. 能够让代码实现更加的规范

而直接用git是无法进行代码的review，或者说是无法提供一套完整的review流程的，因此一般review流程需要在git的基础上进行拓展，而gitlab即已内置review流程功能。

gitlab的review流程分为强制review和非强制review，非强制review仅为强制review的一个子流程，在后续介绍强制review流程中将直接标识出来。

## 准备工作

首先需要一个gitlab服务（废话。。。），至少三个gitlab账号（分别设置好相应的key），账号分别为master（仓库创建者，同时也是最终合并者），reviewer（进行review的对象）与reviewee（被review的对象）

## 创建仓库

由master首先创建出Group，包含reviewer以及reviewee，将reviewer以及reviewee的权限均设置为Developer

由master创建Project，并且将Project归于创建的Group中，并建立出保护分支（master分支默认为proected分支，若需要新建分支，则需要在Project->Settings->Protected Branches设置为保护分支

## 新建分支

reviewee通过master或者需要继承开发的分支新建出一个独立分支，如从master继承出一个为feture_1的分支，修改完后将分支push至gitlab远程仓库中

## 合并申请

reviewee进入gitlab页面，并用reviewee的账号登陆，进入Project对应的设置项，通过Merge Request->NEW MERGE REQUEST，选中需要进行合并的分支，并将Target branch选择为master分支，选择COMPARE BRANCH AND CONTINUE，进入合并申请的设置页，可在Description可@出审核人，将Assignee指定为master，并提交请求

## 检查合并申请

reviewer与master可在Project中看到对应的Merge Request，通过Commit选项卡看到修改处，review结束后看是否需要对该次提醒进行修改，若需要则进行说明注释，若不需要则可直接让master用户在gitlab页面中同意并执行合并，完成代码的review和提交

## PS

reviewee需要自行确保代码运行的正确性，即需要自行对新的feture进行测试，reviewer只需在代码级别进行review。