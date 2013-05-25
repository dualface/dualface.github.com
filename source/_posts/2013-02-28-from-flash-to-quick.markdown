---
layout: post
title: "从 Flash 到 cocos2d-x"
date: 2013-02-28 00:03
comments: true
categories: lua cocos2d quick
---

前言：写这篇文章的原因是朋友公司打算将一个页游产品转为手游，邀请我过去做了两天培训。所以我根据 Flash 团队反映的一些问题做了有针对性的阐述。

Flash 团队转入 cocos2d-x 架构，面对的大部分问题实际上都是“实施”细节，语言根本不是障碍。为了让整个内容更有条理性，我以 Flash 团队提出的问题整理了一个提纲。然后我会按照这个提纲逐步完善文章内容（这次保证不会太监 ^_^）。


阅读索引：

-   cocos2d-x 架构介绍，以及与 Flash 架构的主要差异
-   如何创建、编译、发布 cocos2d-x 项目
-   Lua 与 ActionScript 的主要差异
-   Lua 里如何实现面向对象
-   如何创建一个游戏架构，分离数据、逻辑、表现
-   cocos2d-x 里出现的各种事件，例如触摸、重力感应等
-   cocos2d-x 如何与服务端交互
-   资源的整合与优化
-   画面渲染和内存优化
-   创建自适应多种分辨率的 UI 界面


从 Flash 体系转到 cocos2d-x 在技术上没有太多问题，更主要的门槛还是在工具链和具体实践上。



本文尽可能从 Flash 开发者的角度阐述如何转入 quick 体系。

项目地址： [https://github.com/dualface/quick-cocos2d-x](https://github.com/dualface/quick-cocos2d-x)

<!-- more -->

## 从 Flash 的角度看 quick-cocos2d-x 体系

Flash 实际上是一个工具链，包含了从设计、开发、生成、运行的全套工具。主要由几个部分组成：

-   Flash Professional：制作 Flash 动画的工具
-   Flash Builder：一个基于 Eclipse 的 IDE，以及 Flex SDK
-   Flash Player：在浏览器中运行 SWF 文件的播放器
-   AIR：在桌面和移动设备上运行 SWF 文件的播放器

在开发基于 Flash 的游戏时，不管使用什么框架，最底层都是 Flash Player 提供的一系列 API。这些 API 完成了图像绘制、音乐播放、用户交互等功能。

而 quick-cocos2d-x 则由以下部分组成：

-   quick-cocos2d-x Runtime：游戏引擎
-   quick-cocos2d-x Player：播放器，负责在不同平台上初始化运行环境和游戏引擎，然后载入 Lua 脚本执行
-   tools：一些辅助工具，例如编译 Lua 脚本、创建自定义的 tolua++ 模块等等

<br />

quick-cocos2d-x Runtime 结构：

![from_flash_to_quick_architecture.png](/upload/quick/from_flash_to_quick_architecture.png)


-   最底层是操作系统提供的 OpenGL 库，可以利用硬件加速完成图像绘制（音乐在不同平台会使用不同 API，这里没有单独列出）。
-   中间一层，首先是 C++ 编写的 cocos2d-x 引擎。这个引擎提供一个高性能的游戏开发基础架构，让开发者从不同平台的图像渲染、音乐播放、用户交互等细节中解脱出来。接下来用 tolua++ 这个辅助工具，将 cocos2d-x 引擎的 C++ 接口导入 Lua 环境，让 Lua 脚本可以调用 cocos2d-x 的 API。最后，LuaJIT 提供了一个高性能的 Lua 虚拟机，在支持 JIT 的平台上，可以以接近 C 语言的速度运行 Lua 脚本。
-   Lua Objc/Java Bridge 则是 quick-cocos2d-x 专为游戏集成各种第三方库提供的便捷工具，可以大大降低游戏接入渠道的成本。
-   顶层的 quick framework 是在 cocos2d-x 接口基础上进行的封装，简化了大部分游戏开发时的常用功能。

<br />

虽然 quick 已经提供了 iOS/Android/Windows/Mac 平台的 Player，但如果需要添加自己的第三方库，或者对 quick 做一些扩展，那么就需要安装各个平台自己的开发工具，才能够完成自定义 Player 的创建工作：

![from_flash_to_quick_build_toolchain.png](/upload/quick/from_flash_to_quick_build_toolchain.png)

<br />

## 准备构建环境

> 只有准备自己构建 Player 时才需要准备构建环境。quick-cocos2d-x 提供了功能完善的 Windows 和 Mac 平台下的 Player，可以满足绝大多数情况下的开发需要。具体内容请参考本文下一小节。

第一步自然是获取 quick-cocos2d-x 的源代码。

1.  安装 git 工具，然后下载 quick-cocos2d-x 仓库：

        git clone git://github.com/dualface/quick-cocos2d-x.git

2.  完成后，在 quick-cocos2d-x 目录执行命令（由于要下载几百兆的 cocos2d-x，所以耗时较长）：

        git submodule init
        git submodule update

> 注意：不能使用官方版 cocos2d-x，因为 quick-cocos2d-x 为了加强 Lua 执行，对 cocos2d-x 做了许多改进。

<br />


### 配置 Mac 下的开发环境

准备工作：

1.  从 Apple 网站下载安装最新版 Xcode（已经包含 Mac SDK 和 iOS SDK）
2.  启动 Xcode，选择菜单“Xcode -> Preferences”，打开  Preferences 对话框。切换到 Locations 选项卡，选中“Source Trees”选项卡。并添加一个名为 **QUICK\_COCOS2DX\_ROOT** 的变量，其值为 quick-cocos2d-x 所在目录：

    ![from_flash_to_quick_set_xcode.png](/upload/quick/from_flash_to_quick_set_xcode.png)

3.  进入命令行，输入：

        open -a TextEdit ~/.profile

    添加内容（将其中的 [PATH TO] 替换为实际目录）：

        export QUICK_COCOS2DX_ROOT=/[PATH TO]/quick-cocos2d-x
        export COCOS2DX_ROOT=$QUICK_COCOS2DX_ROOT/lib/cocos2d-x


> 注意：必须使用绝对路径。

设置完成后，打开 **quick-cocos2d-x/samples/CoinFlip/proj.ios/CoinFlip.ios.xcodeproj** 项目，如果编译成功，那么构建 Mac 和 iOS 平台应用的环境就搭建好了。

<br />

1.  从 Google 网站 [http://developer.android.com/sdk/index.html](http://developer.android.com/sdk/index.html) 下载最新的 Android SDK（强烈建议下载 ADT Bundle 版本）。
2.  解压缩以后，运行 tools/android 程序，打开 Android SDK 管理器：

    ![from_flash_to_quick_set_android_mac.png](/upload/quick/from_flash_to_quick_set_android_mac.png)

3.  由于 quick-cocos2d-x 最低支持 Android 2.2 版，所以选中 Android 2.2 SDK，然后下载安装即可。

4.  下载 Android NDK r8b 版本并解压缩（由于 NDK 的特殊性，所以对版本有特定要求），Mac 版下载地址：[http://221.176.14.87/dl.google.com/android/ndk/android-ndk-r8b-darwin-x86.tar.bz2](http://221.176.14.87/dl.google.com/android/ndk/android-ndk-r8b-darwin-x86.tar.bz2)，Windows 版下载地址：[http://221.176.14.96/dl.google.com/android/ndk/android-ndk-r8b-windows.zip](http://221.176.14.96/dl.google.com/android/ndk/android-ndk-r8b-windows.zip)。

5.  再次修改 ~/.profile 文件，加入以下内容：

        export ANDROID_SDK_ROOT=/[PATH TO]/android-sdk-macosx
        export ANDROID_NDK_ROOT=/[PATH TO]/android-ndk-r8b

6.  在命令行模式下进入 **quick-cocos2d-x/samples/CoinFlip/proj.android/** 目录，执行命令 **build_native.sh**，如果最后输出以下内容就表示创建成功：

        SharedLibrary  : libgame.so
        Install        : libgame.so => libs/armeabi/libgame.so

<br />


### 配置 Windows 下的开发配置

quick-cocos2d-x 要求安装 Visual Studio 2012（Express 版或者更高级版本都可以），以及最新的显卡驱动（否则可能出现 Player 无法运行的问题）。

1.  打开系统对话框，添加一个环境变量 **QUICK\_COCOS2DX\_ROOT**，指向 quick-cocos2d-x 所在目录：

    ![from_flash_to_quick_set_windows.png](/upload/quick/from_flash_to_quick_set_windows.png)

2.  打开 quick-cocos2d-x\simulator\proj.win32\LuaHostWin32\LuaHostWin32.sln 工程，进行编译。

<br />

## 用 quick-cocos2d-x 的 Player 打造最简开发环境

只需要从 [https://github.com/dualface/quick-cocos2d-x](https://github.com/dualface/quick-cocos2d-x) 网站直接下载最新版本的 Windows 或 Mac 平台下的 Player 执行文件，我们就可以开始进行游戏开发了。

以 Windows 环境为例：

1.  下载 [https://github.com/dualface/quick-cocos2d-x/archive/master.zip](https://github.com/dualface/quick-cocos2d-x/archive/master.zip)，解压缩后运行 simulator/bin/win32/LuaHostWin32.exe：

    ![from_flash_to_quick_run_luahostwin32.png](/upload/quick/from_flash_to_quick_run_luahostwin32.png)

2.  选择 Player 的菜单“File -> Open Project”，并选择 samples\CoinFlip 为 Project Directory；设置 Screen Size 为 640x960，Screen Direction 为 Landscape：

    ![from_flash_to_quick_run_luahostwin32_project.png](/upload/quick/from_flash_to_quick_run_luahostwin32_project.png)

3.  一切正常的话，点击 Open Project 按钮后就可以看到 CoinFlip 示例程序的画面了：

    ![from_flash_to_quick_run_luahostwin32_ok.png](/upload/quick/from_flash_to_quick_run_luahostwin32_ok.png)

<br />

quick-cocos2d-x 的 Player 提供了创建运行项目、动态切换分辨率、实时调试窗口、一键重启等功能，在开发实践中可以显著提高开发效率。

使用 quick-cocos2d-x 的典型开发流程：

1.  对于一个团队中的大多数开发人员来说，只需要使用 Player 就能完成大部分开发工作。不需要他们安装诸如 Xcode、Visual Studio 等任何开发工具。

2.  如果游戏包含 C/C++ 的扩展代码，由具备完整开发环境的开发人员构建一个自定义 Player ，并分发给整个团队即可。

3.  需要真机测试或发布时，由具备相应开发环境的开发人员编译并输出 .ipa 或 .apk 文件。配合 TestFlight 等第三方 SDK，可以轻松实现错误日志、崩溃记录等功能，为调试提供详细数据。

<br />

- 未完待续 -
