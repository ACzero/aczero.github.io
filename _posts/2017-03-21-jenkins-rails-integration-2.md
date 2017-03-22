---
layout: post
title:  "为Rails项目搭建Jenkins CI服务器(二)"
date:   2017-03-21 22:30:00 +0800
categories: other
---

## 前言

本篇续上一篇<<为Rails项目搭建Jenkins CI服务器(一)>>，将介绍一个实际使用的CI流程。

## Bitbucket+Pipeline实现rails项目CI服务

首先介绍这个CI服务的工作流，先上图：

 ![ci_workflow](/Users/liangzhanhui/Desktop/jenkins2/ci_workflow.png)

 当bitbucket上有新的commit或者pull request时，会通知Jenkins服务器，CI工作流开始。首先是`Preparation`阶段，拉取代码后执行bundle install等命令，为rails项目准备测试环境。接着是`Test`阶段，运行项目对应的测试，输出测试结果。接着是`Report`阶段，可以使用`brakeman`，`rubocop`和`simplecov`等项目检测工具，生成HTML页面报告。最后检测当前分支，若是staging用的分支则部署到staging环境，其他分支则跳过。最后将CI结果返回给bitbucket，要注意的是，只要其中某个stage出错，就会直接返回CI结果。 