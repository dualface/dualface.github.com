---
layout: post
title: "cocos2d-x 2.0 自适应多种分辨率"
date: 2012-08-17 23:17
comments: true
categories: cocos2d
---

cocos2d-x 2.0 提供一个极有价值的新特征: setDesignResolutionSize() 。

这个函数用于指定一个 OpenGL 视图，然后将这个视图映射到设备屏幕上。根据不同的设定，视图会自动缩放显示内容，为 cocos2d-x 自适应多种分辨率提供了基本支持。

<!--more-->

不过要真正实现自适应分辨率，从场景设计、美术制作到程序编写，都需要遵循一套规范，才能极大减少工作量。

{% blockquote %}
注意：本文假定游戏是横向显示的。
{% endblockquote %}

~

## 明确自适应多种分辨率的需求

要让游戏在不同分辨率下都获得良好的用户体验，应该满足这几个要求：

-   背景图填满整个画面，不出现黑边；
-   背景图的主要内容都显示在屏幕上，尽可能少的裁剪图片（减少超出屏幕可视区域的部分）；
-   如果背景图需要放大，尽可能减小放大的比例，避免放大后出现明显的模糊；
-   用户界面的文字标签、按钮在任何分辨率下都应该完整显示，并且容易交互。

上述需求实际上可以分解为两部分：

-   如何制作满足多种分辨率的背景图；
-   如何定位用户界面元素（标签、按钮等）。

~

## 制作适合多种分辨率的背景图

在开始制作背景图前，我们看看市面上各种设备（480 像素分辨率的老设备 2012 的游戏应该可以无视了）常见的像素分辨率（resolution in pixels）：

Device              | Width  | Height
------------------- | ------ | ------
iPad                | 1024px |  768px
New iPad            | 2048px | 1536px
iPhone              |  960px |  640px
Android Phone 1     |  800px |  480px
Android Phone 2     |  854px |  480px
Android Phone 3     | 1280px |  720px
Android Pad 1       | 1024px |  600px
Android Pad 2       | 1280px |  800px

经过几个游戏的实践，我们确定了几个背景图的分辨率：

-   2048px * 1536px

    专门针对 New iPad，设计师的原稿也是这个尺寸。

-   960px * 720px

    这个分辨率针对 iPhone 和 iPad 设备。在 iPhone 上 1:1 显示，上下各剪裁掉 40px。而在 iPad 上按 1.067 放大显示，正好填满整个屏幕，并且用户看不到模糊。从 2048px 的原稿导出 PNG 时，按照 0.469 比例缩小正好就是 960px * 720px。

-   854px * 480px

    市面上的 Android 手机，854px * 480px 和 800px * 480px 是最常见的两种分辨率。2048px 的原稿按照 0.417 比例缩小，然后裁减掉上下多余部分就可以得到需要的 PNG 图片。

-   1280px * 800px

    应付高分辨率的 Android 手机和平板设备，在各种分辨率下都可以获得很好的显示效果。2048px 的原稿按照 0.625 比例缩小，然后裁减掉上下多余部分就可以得到需要的 PNG 图片。

对于美术来说，背景图都按照 2048px * 1536px 的尺寸绘制。然后用脚本配合 ImageMagick 就可以自动导出四种分辨率的背景图片。

如果需要最大程度减小游戏的下载体积，那么可以只使用 960px * 720px 的素材。并且参考本文后面示例程序的做法，用一套素材应付各种不同的分辨率。

唯一需要注意的问题就是：**确保画面中的主要内容在各种设备上都位于屏幕的可视区域中**。

下面几个图展示 2048px 原稿在不同设备上的可视区域：

![](/upload/cocos2d-x-2.0-multires/multires_01.png "")

![](/upload/cocos2d-x-2.0-multires/multires_02.png "")

![](/upload/cocos2d-x-2.0-multires/multires_03.png "")

![](/upload/cocos2d-x-2.0-multires/multires_04.png "")

![](/upload/cocos2d-x-2.0-multires/multires_05.png "")

![](/upload/cocos2d-x-2.0-multires/multires_06.png "")

![](/upload/cocos2d-x-2.0-multires/multires_07.png "")

Photoshop 源文件下载地址: [multires.psd](/upload/cocos2d-x-2.0-multires/multires.psd)

~

## 制作适合各种分辨率的用户界面元素

相比背景图，界面元素的制作只需要考虑一点：**必须能够放置在最小的可视区域中**。如下图界面底部有一排按钮，这些按钮在各种分辨率下都能完整显示：

![](/upload/cocos2d-x-2.0-multires/multires_ui01.png "")

在导出界面元素的 PNG 图片时，仍然使用脚本文件和 ImageMagick 按照特定比例自动缩放。

~

## 在各种分辨率的屏幕上定位界面元素

准备好了美术素材，接下来的挑战就是如何在不同分辨率的设备中定位界面元素。

为了解决这个问题，我们做了大量的尝试，最终找到一种可行的解决方案，而且使用起来非常简单。

### 虚拟分辨率

为了简化程序的开发，我们使用一个统一的虚拟坐标系来映射设备的屏幕。经过几个产品的实践，证明将屏幕宽度设定为 960pt 是很合理的。

**特别注意：在讨论虚拟坐标系时，一律使用 pt（Point）作为单位，而不是 px（Pixel）。**

下面的表格整理了各种设备分辨率与 960pt 宽度虚拟分辨率的对应关系：

Device              | Width  | Height  | Virtual Width | Virutal Height | Scale
------------------- | ------ | ------- | ------------- | -------------- | ------------
iPad                | 1024px |  768px  | 960pt         | 720pt          | 1.066666667
New iPad            | 2048px | 1536px  | 960pt         | 720pt          | 2.133333333
iPhone              |  960px |  640px  | 960pt         | 640pt          | 1.0
Android Phone 1     |  800px |  480px  | 960pt         | 576pt          | 0.833333333
Android Phone 2     |  854px |  480px  | 960pt         | 540pt          | 0.889583333
Android Phone 3     | 1280px |  720px  | 960pt         | 540pt          | 1.333333333
Android Pad 1       | 1024px |  600px  | 960pt         | 562pt          | 1.066666667
Android Pad 2       | 1280px |  800px  | 960pt         | 600pt          | 1.333333333


### 根据参考点定位界面元素

在游戏初始化时，引擎就会根据设备的实际分辨率，自动设定好对应的虚拟分辨率，并且确定屏幕上的几个参考点：

Position            | Value
------------------- | -----------------------
left                | 0pt
right               | 959pt
top                 | 虚拟分辨率的高度 - 1
bottom              | 0pt
center x            | 480pt
center y            | 虚拟分辨率的高度 / 2

有了参考点，定位界面元素就很简单了。例如一个按钮的原点（按钮图片中心点）相对于屏幕左侧 40pt，相对于屏幕底部 30pt。那么在不同分辨率的设备上，这个按钮和屏幕左下角的距离都是差不多的。

只要确保所有界面元素都使用参考点来定位，那么就绝不会出现在设备屏幕上看不到界面元素的情况。

~

## 示例程序

为了方便大家进行测试，本文的示例工程已经编译成 Windows 可执行文件。运行时可以用下列命令行启动以便测试不同分辨率：

``` bash 命令行参数
# 如果没有指定命令行参数，则默认使用 960px * 640px 的屏幕分辨率。
multires.demo1.win32.exe 854 480
```

或者双击 test_multires.cmd 直接查看不同分辨率的运行效果。

其他：

-   示例程序只包含按照 960px * 720px 制作的素材。
-   示例程序可执行文件以及源代码下载：[multires.demo1.win32.zip](/upload/cocos2d-x-2.0-multires/multires.demo1.win32.zip)

~

## 补充说明

本文前面描述了如何创建适合不同分辨率的图片，但最后的示例程序并没有考虑这一点，而是用一套素材就搞定了多种分辨率。实际上，我个人推荐使用一套素材适应多种分辨率，最多再为 New iPad 单独准备一套素材，这样可以显著减少工作量。

如果一定要按照不同分辨率使用不同的素材，那么在显示图片时需要调用 setScale() 调整图片的缩放比例。这样做的原因是 setDesignResolutionSize() 设置虚拟分辨率后，会指定一个全局的缩放比例，所有的图片即便是 scale = 100%，也会自动缩放。所以当图片尺寸和虚拟分辨率不一致时，我们就需要手动调整图片缩放比例了。

假设设备分辨率是 1280px * 720px，虚拟分辨率是 960pt * 540pt，背景图是 1280px * 800px。要确保背景图 1:1 显示在屏幕上，参考如下代码：

``` cpp 让图片按 1:1 显示在屏幕上
const CCSize& winSize = CCDirector::sharedDirector()->getWinSize();
float scale = CCEGLView::sharedOpenGLView().getScaleX();
CCSprite* bg = CCSprite::create("bg.jpg")
bg->setPosition(ccp(winSize.width / 2, winSize.height / 2));
bg->setScale(1.0f / scale); // 这里重置图片缩放比例，确保图片按 1:1 显示在屏幕上
addChild(bg);
```

~

-EOF-
