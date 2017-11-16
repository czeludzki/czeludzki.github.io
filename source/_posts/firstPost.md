---
title: First Post
date: 2017-11-13 16:19:13
tags:
categories: Diary
updated_at: 2017-11-16 10:51:13
---
这个是用 github + hexo 搭建的博客.  

初接触markdown感觉真的很棒,很容易上手,而且功能还很强大!  
另外我们的主角hexo还真的是简约不简单,轻便是当然的了. 可惜写个博客还要必须用电脑终端提交一下的,确实有点跟移动终端时代脱节了,不过也是足够装逼的😆😆可谓双刃剑吧.哈哈
<!-- more -->
搭建工作方面主要是参考了
[hexo官方文档](https://hexo.io/zh-cn/docs/) 以及 [HEXO+Github,搭建属于自己的博客](http://www.jianshu.com/p/465830080ea9).  

在博客的代码管理上,主要是参考了知乎上这位大神 使用两个分支分别管理 public代码 以及 source代码 的方案  
> [使用hexo，如果换了电脑怎么更新博客？ - CrazyMilk的回答 - 知乎](https://www.zhihu.com/question/21193762/answer/79109280)

问题来了, 我的所用的 next 主题是通过 `git clong 主题.git themes/next` 的方式拉下来的, 尽管 `/.git/`、`/themes/`、 `/source/` 是同级文件夹, 但是我无论怎么 commit 代码都无法将主题 `/themes/` 也一并提交到 source 的管理分支(抱歉对git的深层原理实在是小白,所以无法解释当中原由).  

这样将导致我换台电脑写下blog, 对主题文件的配置修改都会丢失, 不科学!  

一轮google,终于找到了解决方案...  

回到github,找到我想要的主题 [hexo-theme-next](https://github.com/iissnan/hexo-theme-next.git), 右上角, fork它, 然后回到自己 github 的 Repositories, 找到刚 forked 的 主题, `git clong https://github.com/你的githud名/hexo-theme-next.git theme/next`.  
然后你爱怎么改, 放心 commit & push, 换台电脑以后, 除了要 pull source 的代码, 还要 pull 一下 主题的代码.
终端依次执行 `hexo g` `hexo s` 打开看看, 主题配置还在👌.

最后附上我最近在公司用的桌面,太美了有没有.在百度找的,找不到原地址了  
![img](http://ozej1b09v.bkt.clouddn.com/%E5%9C%9F%E6%98%9F0.jpg)  
对了, 图片是存在 [七牛云](https://www.qiniu.com/) 上,已认证的免费用户享受10G空间以及每月10G国内及10G国外流量.

------
### 17.11.16补充
在执行 hexo deploy 时出现权限错误,解决参考
[Error- Permission denied (publickey).错误详解](http://www.wangnunu.com/2016/04/04/Error-Permission-denied-publickey-%E9%94%99%E8%AF%AF%E8%AF%A6%E8%A7%A3/)
