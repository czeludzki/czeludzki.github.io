---
title: '集成了友盟app分析,debug时无法打印崩溃原因的解决'
date: 2019-04-02 17:44:39
tags:
categories:
---
继承有梦数据分析以后, 所有的崩溃都没有打印, 烦, 百度了好久, 大多都说只要设置  
`[MobClick setCrashReportEnabled:NO];`  
问题就能解决的了.  
并没有啊!!!!  
去友盟官网找了一下, 从它们的文档中找到了关键的一句:

``` objc
[MobClick setCrashReportEnabled:NO];   // 关闭Crash收集
// **注意：**
// 此函数需在common sdk初始化前适用，默认是开启状态
```

好吧, 试了一下, 把本来的代码:
```objc
[UMConfigure initWithAppkey:KUMengAppKey channel:nil];
[MobClick setCrashReportEnabled:!IS_DEBUG];
```
改为:
```objc
[MobClick setCrashReportEnabled:!IS_DEBUG]; // 这一行在 sdk 初始化前执行
[UMConfigure initWithAppkey:KUMengAppKey channel:nil];  // 初始化友盟sdk
```

OK, 奔溃也有详细的打印了.