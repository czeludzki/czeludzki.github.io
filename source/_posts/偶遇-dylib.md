---
title: 偶遇.dylib
date: 2018-04-24 21:46:12
tags:
categories:
---
>以下所说的并不是 ffmpeg 引用到项目中的正确方法, 请不要模仿.  

### 一切的一切都要从编译 ffmpeg, 并在 mac os 项目上引用 ffmpeg 说起.  
<!-- more -->
我是用的 homebrew 装的 ffmpeg, 路径是 `/usr/local/Cellar` 里面发挥主要用处的有
`bin lib include` 三个文件, `bin` 存放有 ffmpeg, ffprobe 等可执行文件, `include` 都是头文件, `lib` 里面放的就是我所偶遇的一大堆 `.dylib` 以及 `.a` 库文件.  
![](WechatIMG3.jpeg)

###### 因为想要做个 ffmpeg 给视频转码的项目, 要将 ffmpeg 引入, 在引入这个阶段, 我走了捷径, 可以说是搭上了穿梭机(其实是特傻😂).  
我直接从 `/usr/local/Cellar` 把 `lib` `include` 两个文件拉到项目中, 在项目中用到了 `avformat` `avcodec` 等等的类, 一切看起来都非常的顺利, 直到我从 ffmpeg 3.4 转到了 4.0 版本...  
我通过 `brew update` && `$ brew reinstall ffmpeg --with-x265` 命令将 ffmpeg 升级到 4.0版本, 再打开项目, 一运行就报如下错误:
```
dyld: Library not loaded: /usr/local/opt/ffmpeg/lib/libavdevice.58.dylib  
Reason: image not found
```
![](WechatIMG5965.jpeg)

思考一下, 很大的原因是出在这一堆 `.dylib` 库中. 报错也让我感到很奇怪, 明明已经将这一堆 `.dylib` 放到项目中, 怎么 `not loaded` 的路径会是 `/usr/local/opt/ffmpeg/lib/libavdevice.58.dylib` ??  

我到 `DerivedData` 文件夹中找到了项目生成的 `.app` 文件, 发现里面并没有 Frameworks 文件夹也找不到任何打包进去 `.app` 的 `.dylib` 库.  
据我所知由于apple为了禁止热更新的关系, 动态库只能通过 `Embedded binaries` 嵌入式动态库打入到 `.app` 中, 我一拍大腿以为自己已经找到了解决问题的办法, 赶紧将所有 `.dylib` 文件加入到`Embedded binaries`中.   
![](WechatIMG4.jpeg)
##### 然而并没有什么撚用...
我内心一直在认为, 这个 `.dylib` 不简单, 里面一定有什么使app在运行的时候指向那个我不想指向的地方: `/usr/local/opt/ffmpeg/`.  
在一番google过后, 这两篇文章帮了我的大忙:  
>[MacOS平台下@rpath在动态链接库中的应用](https://www.cnblogs.com/csuftzzk/p/mac_run_path.html)  
>[Mac下动态库的install_name](https://blog.csdn.net/vincent2610/article/details/56839494)  

其中文章 [Mac下动态库的install_name](https://blog.csdn.net/vincent2610/article/details/56839494) 详细地指出了 `install name` 在 `.dylib` 的作用.
我也用 `otool -D /我的app包/content/Frameworks/libavdevice.58.dylib` 查看了 `libavdevice.58.dylib` 的 `install name`, 是指向着 `/usr/local/opt/ffmpeg/lib/libavdevice.58.dylib`, 这也解释了为什么上面的报错路径会是这个.  
那么解决问题的办法就是, 使 `.dylib` 包的 install name 指向正确的路径(通俗解释, 如有不对感谢指出).  
以其中一个 `.dylib` 为例, 使用命令
```
$ install_name_tool -id @rpath/libavdevice.58.dylib 我的项目路径/lib/libavdevice.58.dylib
```
就将 `libavdevice.58.dylib` 的 `install name` 修改为 `@rpath/libavdevice.58.dylib`, 而且这个以 `@rpath` 修饰的 `install name` 针对各个引用这个 `.dylib` 库的项目而言是 `动态的` && `是由项目的 run path 决定的`, 对于 `@rpath` `@loader_path` `@executable_path` 的区别, 具体请看 [MacOS平台下@rpath在动态链接库中的应用](https://www.cnblogs.com/csuftzzk/p/mac_run_path.html).

上面提到了
> 以 `@rpath` 修饰的 `install name` `是由项目的 run path 决定的`  

那么我还需要在项目里 `Build Settings` 中 `Runpath search paths` 里面加入我将决定这些引入的 `.dylib` 库真正完整的 `install name`  
![](WechatIMG5.jpeg)

## 还有更恶心人的,
## `lib` 文件夹里面有十几个这样的 `.dylib` 库, 这意味着, 我要重复这个 `$ install_name_tool -id @rpath/libavdevice.58.dylib 我的项目路径/lib/libavdevice.58.dylib` 命令十几次...

>我还一度想要写个 python 脚本来完成执行这个命令十几次的任务, 当然我傻, 但是并没至于如此. 静态编译 ffmpeg 才是出路, 当然这也是后话了.

秉承着 `实践是检验真理的唯一道路` 这一深深印在我心中的理念, 我手动..人肉..亲自执行了这个命令十几次, 将在项目中 `lib` 文件夹下的所有的 `.dylib` 的 `install name` 都执行了一次, 并且 `brew uninstall ffmpeg` 删除了本地的 ffmpeg 以确保不会干扰到我这个无聊的实验.

运行项目, 不再报错, ffmpeg使用正常, 使用 `otool -D` 命令检查 .app 包包里 /Frameworks 里面 `.dylib` 库的 `install name` 确实指向了如我所愿的位置.  
DONE..

能够认识到 `.dylib`, 人肉执行十几次那个修改 `install name` 的命令, 也是值得.
