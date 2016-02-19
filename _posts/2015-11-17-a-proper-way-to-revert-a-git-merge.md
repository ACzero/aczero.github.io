---
layout: post
title:  "Git撤销某次merge的正确实现方法"
date:   2015-11-17 20:29:51 +0800
categories: git
---

## 场景
使用rails开发新功能，到了前后端集成的阶段，都在一个branch上工作。前端那边有一段时间没pull代码了，就把我后端的修改拉下来merge，而然前端并不是很熟悉Git,因此我后端的修改莫名的在那次merge中消失了......情况如下:
![my situation]({{ site.url }}/assets/2015-11-17_1.png)
红圈部分就是出问题的那次merge，另一个前端不知道这个情况，提交了代码之后也把这个修改merge了过来，因此现在remote的HEAD已经把我后端这边的修改弃掉了....

> 临时的解决方案

我打算使用revert来撤销这次merge的修改，查阅revert的文档：

<b>-m parent-number</b>

<b>--mainline parent-number</b>

Usually you cannot revert a merge because you do not know which side of the merge should be considered the mainline. This option specifies the parent number (starting from 1) of the mainline and allows revert to reverse the change relative to the specified parent.

Reverting a merge commit declares that you will never want the tree changes brought in by the merge. As a result, later merges will only bring in tree changes introduced by commits that are not ancestors of the previously reverted merge. This may or may not be what you want.

大意为:通常情况下revert只能回滚某次提交，但是通过添加`-m`选项就可以回滚某次merge，`-m`需要提供一个参数`parent-number` `(1或2)`用来指定主分支，下面来说明一下`revert -m parent-number <commit>`是如何工作的:

    remote: o---a---b---HEAD
             \     /   /
    local:    N---M---

前端在本地把远端的修改merge下来，出问题的就是上图中的M处提交(没有把a,b的修改merge进去)，然后前端又把修改push到了远端，现在远端的HEAD已经丢了a,b的修改。

所以我需要撤销M这个地方问题就是parent-number应该选择哪个?在这里，是本地去merge远端分支，因此1代表local，2代表remote，选择的parent会被保留，而另外一个会全部被撤销(包括前面的提交)。毫无疑问，我要保留的是远端的修改，因此使用命令是`git revert -m 2 <commit>`，这样a,b的修改就回来了，而N处的修改就被撤销掉。(可以让前端reset回去，然后再merge一次)

## 然而Revert并没有这么简单
以下内容译自:[how to revert a faulty merge](https://github.com/git/git/blob/master/Documentation/howto/revert-a-faulty-merge.txt)，稍微有点简化，这篇文章讨论的是撤销不同的分支之间的merge。(跟我遇到的场景略有不同，我是在同一个分支上)

>revert没有按我们想的去工作?


场景：某程序员在分支`new-func`上做新功能，并称功能已经完成，我们便将其merge到`master`(M处)，继续工作了一段时间我们发现A和B的功能存在BUG，导致我们整体功能都不正常了，所以我们自然要revert掉这次merge，W代表revert操作。这是现在我们分支的情况：


     master:  ---o---o---o---M---x---x---W
	                        /
     new-func:      ---A---B

程序员回去修复分支上的bug，工作了一段时间，我们的分支是这样的情况：

     master:  ---o---o---o---M---x---x---W---x
	                        /
     new-func:      ---A---B-------------------C---D

C和D修复了A和B中存在的bug，赶紧把分支merge到`master`。不幸的是，当我们再次把`new-func`merge到`master`后，只有C和D的修改存在，而A和B的修改都不存在。而导致这个问题的正式W的revert操作。

> revert到底做了什么?

在上面的情景中，当我们使用以下命令`git revert W`,A和B的修改又被找回来了：

     master: ---o---o---o---M---x---x---W---x---Y
	                       /
     new-func:     ---A---B-------------------C---D

我们在Y处revert的是W，是某次提交，这样我们的W和Y都像不存在一样：

     master: ---o---o---o---M---x---x-------x----
	                       /
     new-func:     ---A---B-------------------C---D

当我们再去merge的时候，看来起就是这样的：

     master: ---o---o---o---M---x---x-------x-------*
	                       /                       /
     new-func:     ---A---B-------------------C---D

当我们revert的是某次commit的时候，那么那次提交在历史上就好像是不存在一样，这就是我们想要的。

然而，当我们revert的是某次merge的话，实际上这次merge依然存在于我们的历史中，而我们只是把他的修改(A,B)给撤销掉了，所以，当我们再次去merge`new-func`的时候，由于W的存在，A，B依然会被撤销掉。

对于revert我们应该这样理解：revert做的工作并不是回滚(想要回滚我们可以使用reset)，而是把某次的修改还原，等于帮我们把那次修改的文件改回来，再做一次commit。

>更合理的解决方法

有的时候，重复revert并不是个好的解决方法。

首先我们有这样的两个分支：

     P---o---o---M---x---x---W---x
      \         /
       A---B---C

我们出现问题的提交是B，因此我们需要有A中的修改，一般情况下我们会checkout回去A，并使用`rebase -i P`去修改B，我们的分支变成这样：

     P---o---o---M---x---x---W---x
      \         /
       A---B---C   <-- 旧分支
        \
         B'---C'   <-- 重写的分支

当我们把重写的分支merge回去`master`的时候，我们发现A的修改依然会被W给revert掉...T_T

其实我们有更好的做法，让我们来创建一个新的分支：使用`$ git rebase [-i] --no-ff P`命令，`--no-ff`这个选项让我们创建一个全新的分支(所有提交的SHA ID都不同于原来)，我们的分支变成这样：

       A'---B'---C'  <-- 全新的分支
      /
     P---o---o---M---x---x---W---x
      \         /
       A---B---C


这样我们merge回去，A也依然存在：

       A'---B'---C'------------------
      /                              \
     P---o---o---M---x---x---W---x---M2
      \         /
       A---B---C


好了，接下来让我们回到开始的问题，我们的情况是这样的：

     P---o---o---M---x---x---W---x
      \         /
       A---B---C----------------D---E   <-- 修改后

我们使用以下两条命令

 * `git checkout E`
 * `git rebase --no-ff P`

变成这样：

       A'---B'---C'------------D'---E'  <-- 全新的分支
      /
     P---o---o---M---x---x---W---x
      \         /
       A---B---C----------------D---E

这样我们就可以把修改merge进去`master`了：

       A'---B'---C'------------D'---E'
      /                              \
     P---o---o---M---x---x---W---x---M2
      \         /
       A---B---C

## 最后

其实看完之后我发现跟我遇到的情况不相同，不过这种做法还是值得记下来的，对于我这种场景，我认为我的解决方法是正确的，但是或者有更好的解决方法，欢迎指出~~
