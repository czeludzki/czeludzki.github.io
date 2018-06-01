---
title: 记录一下ffmpeg的静态编译以及如何在项目中以单进程异步线程执行ffmpeg命令
date: 2018-05-30 20:39:00
tags: ffmpeg
categories:
---

上一篇文章也有提到, 将动态库加入到项目中使用实在并不是上策, 所以有了这个静态库编译并在项目中使用的实践记录.  
可能网上很多地方都有相似的教程和说明, 所以在这里我打算尽可能说一些别人没说过的.(其实是我找不到的😂)  
<!-- more -->

开始之前说明一下为什么我认为将ffmpeg编译的动态库加入到项目中并不是上策:
* 包会相对的大.
* 如上一篇所说, 要对编译出来的可执行文件进行重定向 rpath 操作. 动态包太多导致过程太过繁琐.
* 由于我的需求主要是以 ffmpeg 做转换操作, 希望可以在界面上实时地看到转换过程, 是否出错, 可以好像使用 Terminal 一样, 对错误的行有针对性的颜色显示.  如图  ![](WechatIMG1450.jpeg)  
而如果使用 NSTask 执行 ffmpeg, 我并不能通过 NSTask 准确地捕获到 ffmpeg 执行期间每行输出命令的level.  
这里说的 level 是 ffmpeg 中 `libavutil/log.h` 中定义的一系列宏定义, 包括 `AV_LOG_ERROR`, `AV_LOG_WARNING`, `AV_LOG_VERBOSE` 等等.
而如果我直接用 `ffmpeg.c` 中的 `main()` 函数执行 ffmpeg 命令, 我不仅可以准确地捕获每行输出的 level, 还可以通过 `main()` 函数的返回, 判断到整个过程是否顺利完成.

# A.
编译过程不再赘述, 值得注意的是编译的配置
```objc
--prefix=/Users/username/targetPath --disable-doc --enable-pic --enable-cross-compile --enable-pthreads --enable-version3 --enable-nonfree --disable-shared --enable-static --enable-hardcoded-tables --cc=clang --host-cflags= --host-ldflags= --enable-gpl --enable-libfdk-aac --enable-libmp3lame --enable-libx264 --enable-libx265 --enable-libxvid --enable-opencl --enable-videotoolbox --enable-audiotoolbox --disable-lzma
```

* 其中 `--disable-shared` 和 `--enable-static` 分别是指: 不编译为动态库, 编译为静态库  
* `--disable-programs` 是指不生成可执行文件(ffmpeg, ffprobe, ffplay 等)

# B.
将编译出来的两个文件夹, `include`,`lib` 加入项目, 并加入相关依赖库, 编译通过.  
相关依赖库如图  
![](WechatIMG1453.jpeg)  
这里要注意的是, 记得加入你需要的第三方依赖库, 不然编译会报错, 就如上面编译配置中我加入了 `--enable-libx265`, 那么就去libx265的开发官网下他们的 libx265.a.  
不过我不是去他们官网下载的, 哈哈哈哈, 我是用的 homebrew 在电脑上另外装了 ffmpeg, homebrew 将 `下载ffmpeg`, `编译ffmpeg`, `下载相关依赖库`, `搭建ffmpeg环境` 四大工作都做好了, 如无意外可以在 `/usr/local/Cellar` 文件夹下找到, 找到后将相关的 .a 文件加入到项目即可.

# C.
将 ffmpeg 源代码中的 fftools 中以下一坨文件加入项目:
>* ffmpeg.c
* ffmpeg.h
* cmdutils.c
* cmdutils.h
* ffmpeg_cuvid.c
* ffmpeg_filter.code
* ffmpeg_hw.c
* ffmpeg_opt.c
* ffmpeg_videotoolbox.c  

编译, 然后是一定会报错的. 各种找不到 `#include "谁谁谁"` , 好可怕对吧. 这些报错真是把我吓尿了, 后来我大概是做了这些事:
* 尽可能不对 找不到的 `#include xxx` 做 注释/删除, 因为可能会涉及更多依赖定义找不到, 找不到的话就去 ffmpeg 的源码中去拉取.  
例如 `#include "libavutil/thread"` 说找不到, 那就去 ffmpeg 源码 中去拉. `#include "config.h"` 文件找不到, 我就去 ffmpeg 源码去拉, 反正想要的都在 ffmpeg 的源码里.  
如图  
![](WechatIMG1455.jpeg)  
* 我还记得我编译出来的 `lib` 文件夹中并没有 `libswresample` 相关的文件夹, 也就是意味着, 我编译出来要用的功能是用不着 `libswresample` 这个模块内容的, 那么我就直接把所有说找不到 `#include "libswresample/xxxx"` 的都 注释/删除 掉, 甚至将代码中实际使用到的相关报错都注释掉.
* 反正大胆操作就对了. 要不就拖到项目, 不行就注释掉.
* 如果最后不再跟你说找不到 `#include "xxxxx"` 了, 到了链接那一步出错, 那几乎 80% 的情况是少了某些依赖库.  
我记得在我步骤 B 中是顺利编译的, 到了步骤 C, 加入了 `fftools` 文件夹以下的那坨文件后, 在链接状态中报错, 隐约记得是 `"_clxxxxxx"` 方法找不到了, 那就加入了 `OpenCL.framework`, 就好了.
* 如果各种库加上了, 编译时依然是在链接的那一步有问题, 我偷偷告诉你, `ffmpeg.c` 中有个 main() 函数, 一个项目中(可执行文件)有且只能存在一个 main() 函数. 把 `ffmpeg.c` 中的 mian() 函数改名, 例如 ffmpeg_main() 之类的.  

# D.
走过了ABC后, 编译顺利通过, 就是成功了一半了. 接下来就是用 ffmpeg_main() 函数执行命令.  
```objc
- (void)executeCommand:(NSArray<NSString *> *)commands
{
    int argc = (int)commands.count;
    char** argv=(char**)malloc(sizeof(char*)*argc);

    for(int i=0; i<argc; i++){
        argv[i] = (char*)malloc(sizeof(char)*1024);
        strcpy(argv[i],[[commands objectAtIndex:i] UTF8String]);
    }

    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        int rec = ffmpeg_main(argc, argv);
        for (int i=0;i<argc;i++) free(argv[i]);
        free(argv);
        NSLog(@"rec = %i",rec);
    });
}
```
我就是这么干的.  
如果你直接这么干就出问题了. 你会发现命令确实可以跑起来, 但是完毕了, app就自动退出.
问题出在 `cmdutils.c` 中的 `exit_program()` 函数, 其中的 `exit(ret)` 一行.  
ffmpeg 每次执行命令时, 都会创建一个新的进程去执行, 当命令执行完毕, 就会调 `exit_program()` 做一系列的清理工作, 然后 `exit(ret)` 直接把进程杀死.  
macos以及iOS的app都是一般来说都是以单进程的设计概念为主, 尤其是非越狱的 iOS 更是没有多进程的概念, 这也能解释了, 为什么当命令执行完毕, app就自动退出.  

那就先注释掉 `exit(ret)` 这一行, 使命令执行完毕, 也不用杀掉进程.
再次运行命令 `ffmpeg`(就这个简单的命令而已) , 出错, 依稀记得是 访问了某个空数组越界的错误.
分析一下 ffmpeg.c 中被改名为 ffmpeg_main() 的 main() 函数,  
如图:  
![](1527693870904.jpg)  
由于我要求执行的命令只是简单的 `ffmpeg`, 所以函数执行到上图中 `exit_program(1)` 处就应该结束, 如果 `exit()` 那一行代码没有被注释掉, 那么, 根本不可能继续执行下去.那么原因大概就知道了.    
所以, 我将 `exit_program()` 函数修改了一下(头文件和实现都要修改), 改为:
```objc
int exit_program(int ret)
{
    if (program_exit)
        program_exit(ret);
//    exit(ret);
    return ret;
}
```
并且在 ffmpeg() 函数中的所有 `exit_program(xxx)` 都改为 `return exit_program(xxx)`, 使 ffmpeg() 函数执行到 `exit_program(xxx)` 直接结束, 不再往下走.

#### 但是问题依旧!!
即使有了 `return exit_program(xxx)`, 函数还是继续往下走, 百度一下, 找到了问题出现的原因.  
![](1527694497040.jpg)  
原来是 `cmdutils.h` 文件中 `exit_program()` 函数末尾的 `av_noreturn` 修饰导致 `exit_program()` 函数被合法地认定为没有返回, 所以即使在外部调用 `return exit_program(xxx)`, 都无法正常地退出函数.  
`av_noreturn` 的详细解释请参考:  

>[__ATTRIBUTE__ 你知多少？](https://www.cnblogs.com/astwish/p/3460618.html)   

那么几乎可以认定, 这个 `av_noreturn` 修饰, 只是为了 `exit()` 函数存在而服务的, 现在 `exit()` 不存在了, 那我就大胆把它删掉了.
后来还根据雷神的提示, 在 `ffmpeg.c` 中 `ffmpeg_cleanup()` 函数的末尾 加上各种变量的清零操作:  
```objc
nb_frames_dup = 0;
nb_frames_drop = 0;
nb_filtergraphs = 0;
nb_input_files = 0;
nb_input_streams = 0;
nb_output_files = 0;
nb_output_streams = 0;
```
然后顺利运行.

# E.
上面提到了, 我需要准确地捕获到每一个 ffmpeg 的输出日志, 并且可以好像 Terminal 执行 ffmpeg 一样输出的日志根据不同的log_level显示不同的颜色.  
具体可以去看看 ffmpeg 源码中 `libavutil/log.c` 是怎么实现 `av_log()` 函数的, 这里简单说一下, 大概就是使用 `vsnprintf()` 组合一下输出的字符串, 然后再根据函数的参数 `int level` 设置输出的字体颜色.  
如果我们要插手 `av_log` 方法来达到需求, 那不仅仅麻烦, 还更进一步地破坏了 ffmpeg 的原生代码, 其实, ffmpeg 提供了一个 `av_log_set_callback(void (*callback)(void*, int, const char*, va_list))` 函数, 只要传入一个 callback 函数的地址, 那么该函数就会接受所有 av_log 的回调.  

```objc
void FFLogCallBackFunc(void *ptr, int level, const char *fmt, va_list vl)
{
    char o[1024];
    vsnprintf(o, 1024, fmt, vl);
    NSString *output_desc = [NSString stringWithUTF8String:o];
    NSLog(@"output_desc = %@\nlevel = %i",output_desc,level);
}

int main(int argc, const char * argv[]) {
    av_log_set_callback(&FFLogCallBackFunc);
    return 0;
}

```


# F.
目前为止, 执行媒体格式重新封装, 跨文件封装新的媒体文件, 对音视频字幕流的转码, 对视音频字幕流重新配置一些基础的参数例如采样率比特率视频的scale等都没有任何问题. 如遇到任何问题, 就在这里继续 GHIJK...
