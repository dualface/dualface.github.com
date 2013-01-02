---
layout: post
title: "LuaJavaBridge - Lua 与 Java 互操作的简单解决方案"
date: 2013-01-01 01:01
comments: true
categories: lua, java
toc: true
---

最近在游戏里要集成中国移动的 SDK，而这些 SDK 都是用 Java 编写的。由于我们整个游戏都是使用 Lua 开发的，所以就面对 Lua 与 Java 互操作的问题。

传统做法是先用 C/C++ 借助 JNI（Java Native Interface）编写调用 Java 的接口函数，然后再将这些函数通过 tolua++ 导出给 Lua 使用。这种做法最大的问题就是太繁琐，而且稍微有一点点修改，就要重新编译，严重降低了开发效率。

我尝试写了几个接口函数后，发现 JNI 提供了完善的接口来操作 Java，比如查找特定的 Class、Method 等等。既然有这些东西，我想完全可以实现一个很薄的转接层。这个层会提供一些函数，让 Lua 代码可以直接调用到 Java 的方法。

经过一番努力，LuaJavaBridge（简称 luaj）诞生了。

<!-- more -->


## luaj 主要特征 ##

-	可以从 Lua 调用 Java Class Static Method
-	调用 Java 方法时，支持 int/float/boolean/String 四种参数类型
-	支持以用数组的形式，批量传递数据到 Java
-	可以将 Lua function 作为参数传递给 Java，并让 Java 保存 Lua function 的引用
-	可以从 Java 调用 Lua 的全局函数，或者调用引用指向的 Lua function

luaj 的功能很简单，但对于集成各种 SDK 来说已经完全满足需求了。


## luaj 用法示例 ##

下面的代码是我们游戏中实际使用的中国移动支付 SDK 调用代码，luaj 好不好用一目了然：

**Lua 代码:**

``` lua
--[[
购买 1000 金币

Java 方法原型:
public static void GameInterface_doBilling(final String billingIndex,
        final boolean useSms,
        final boolean isRepeated,
        final int luaFunctionId)
]]

-- 用于处理支付结果的函数
local function callback(result)
    if result == "success" then
        game.state:increaseCoins(1000)
        game.state:save()
    end
end

-- 调用 Java 方法需要的参数
local args = {
    "001",          -- billingIndex
    true,           -- useSms
    true,           -- isRepeated
    callback        -- luaFunctionId
}
-- Java 类的名称
local className = "com/qeeplay/frameworks/ChinaMobile_SDK"
-- 调用 Java 方法
luaj.callStaticMethod(className, "GameInterface_doBilling", args)
```

上面的代码就不解释了，注释已经写得非常明白。


## luaj 实现原理 ##

luaj 的核心目标有两个：从 Lua 调用 Java, 从 Java 调用 Lua。整理出来就是如下几点：

-   查找并调用指定的 Java 方法
-   检查调用结果，并从 Java 方法获取返回值
-   将 Lua function 作为参数传递给 Java 方法
-   在 Java 方法中调用 Lua function


### 查找并调用指定的 Java 方法 ###

JNI 提供了 FindClass() 方法用于查找指定的 Class，所以 luaj.callStaticMethod() 的第一个参数就是要调用的 Java Class 的完整类名称（类名称中的“.”要替换为“/”）。

找到指定 Class 后，利用 JNI 的 GetStaticMethodID() 方法就可以找到这个类的指定静态方法，前提是要提供静态方法的名称和签名。

所谓签名，就是指 Java 方法的参数类型和返回类型定义。例如前面示例代码中 GameInterface\_doBilling() 方法的签名是 (Ljava/lang/String;ZZI)V 。关于 Java 方法签名的具体定义，可以参考：[JNI Type Signatures](http://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/types.html#wp16432)。

由于签名有点难懂，而且和 Lua 的类型定义不一致，所以 luaj 可以根据调用参数自动猜测方法签名。示例代码中，luaj.callStaticMethod() 的第二个参数指定了要查找的方法名称，但并没有提供方法的签名，这就是利用了 luaj 的自动猜测签名功能。

示例代码指定参数部分，一共指定了 4 个参数，分别是：字符串、布尔值、布尔值、Lua function。

``` lua
-- 调用 Java 方法需要的参数
local args = {
    "001",          -- billingIndex
    true,           -- useSms
    true,           -- isRepeated
    callback        -- luaFunctionId
}
```

luaj 根据这 4 个参数，会创建出 GameInterface\_doBilling() 方法的签名。*注意 Lua function 是以整数的形式传入 Java 方法，所以 Java 方法的第四个参数是 int 类型）。*

不幸的是 Lua 里没有办法准确判断一个数值是整数还是浮点数，所以 luaj 假定所有的数值都是浮点数。下面的代码第二个调用就会失败：

``` lua
local args = {1} -- 生成的方法签名是 (F)V

--[[
Java 方法原型:
public static void TestMethod1(final float integerValue)
]]
-- 调用成功
luaj.callStaticMethod(className, "TestMethod1", args)

--[[
Java 方法原型:
public static void TestMethod2(final int integerValue)
]]
-- 调用失败，正确的方法签名应该是 (I)V
luaj.callStaticMethod(className, "TestMethod2", args)
```

为此，luaj 允许开发者指定完整的方法签名。而且除了整数和浮点数的情况，在需要从 Java 方法获得返回值时，也需要开发者指定完整的方法签名。示例代码如下：

``` lua

local args ={"StringValue", 1, 3.14}

--[[
Java 方法原型:
public static int TestMethod3(final String stringValue,
        final int integerValue,
        final float floatValue)
]]

-- 定义签名
-- 返回值: [I]nt
-- 参数: [S]tring, [I]nteger, [F]loat
local sig = "I(SIF)"

-- 调用方法并获得返回值
local ok, ret = luaj.callStaticMethod(className, "TestMethod3", args, sig)
```

为了尽量与 Lua 的习惯保持一致，所以 luaj 指定的方法签名与 Java 的要求有些区别，luaj 会自动完成转换。但 luaj 也支持 Java 风格的方法签名，与上面一段代码功能等同的签名定义如下：

``` lua
local sig = "(Ljava/lang/String;IF)I"
```

~


### 检查调用结果，并从 Java 方法获取返回值 ###

luaj 调用 Java 方法时，可能会出现各种错误，因此 luaj 提供了一种机制让 Lua 调用代码可以确定 Java 方法是否成功调用。

luaj.callStaticMethod() 会返回两个值：

-   当成功时，第一个值为 true，第二个返回值是 Java 方法的返回值（如果有）。
-   当失败时，第一个值为 false，第二个返回值是错误信息。

下面的代码展示了如何检查返回结果和获得返回值：

``` java Java 代码
public static int AddTwoNumbers(final int number1,
        final int number2) {
    return number1 + number2;
}
```

``` lua Lua 代码
local args = {2, 3}
local sig = "I(II)"
local ok, ret = luaj.callStaticMethod(className, "AddTwoNumbers", args, sig)

if not ok then
    print("luaj error:", ret)
else
    print("ret:", ret) -- 输出 ret: 5
end
```

~

目前，luaj 支持从 Java 方法返回 4 种基本值类型：整数、浮点数、布尔值、字符串。

~


### 将 Lua function 作为参数传递给 Java 方法 ###

很多时候，我们需要一种方法让 Java 代码可以向 Lua 代码传递一些消息。例如在大部分游戏平台的 SDK 中，涉及支付的部分都是异步操作的。在支付操作结束后，Java 代码需要通知 Lua 支付成功与否。

Lua 虚拟机中，Lua function 以值的形式保存。但这个值无法直接给 Java 用，所以 luaj 做了一个 Lua function 引用表。当一个 Lua function 传递给 Java 时，这个 function 对应的值会被存在引用表中，并获得一个唯一的引用 ID （整数）。Java 代码拿到这个引用 ID 后，就可以很方便的调用该 Lua function 了。

回顾最开始的示例代码，GameInterface\_doBilling() 函数用于接收 Lua function 的参数就是 int 类型。因为实际传入 Java 函数的值是 Lua function 的引用 Id。

~


### 在 Java 方法中调用 Lua function ###

在 Java 代码中拿到 Lua function 的引用 ID 后，就可以很方便的调用该 Lua function 了：

``` java
LuaJavaBridge.callLuaFunctionWithString(luaFunctionId, "hello");
```

这里出现的 LuaJavaBridge 是 luaj 的 Java 部分定义的工具 class。其中定义的 callLuaFunctionWithString() 方法可以将一个字符串参数传递给指定的 Lua function。

LuaJavaBridge 还提供了 callLuaGlobalFunctionWithString() 方法，可以直接调用 Lua 中指定名字的全局函数。这样可以在没有 Lua function 引用 ID 的情况下和 Lua 代码交互。

由于自己的项目暂时没更多需求，所以目前 luaj 只支持向 Lua function 传递单个字符串参数。

~


## GL 线程和 UI 线程的协调 ##

cocos2d-x for Android 运行在多线程环境下，所以在 Lua 和 Java 交互时需要注意选择适当的线程。

~

cocos2d-x 在 Android 上以两个线程来运行，分别是负责图像渲染的 GL 线程和负责 Android 系统用户界面的 UI 线程。

-   在 cocos2d-x 启动后，Lua 代码将由 GL 线程调用，因此从 Lua 中调用的 Java 方法如果涉及到系统用户界面的显示、更新操作，那么就必须让这部分代码切换到 UI 线程上去运行。
-   反之亦然，从 Java 调用 Lua 代码时，需要让这个调用在 GL 线程上执行，否则 Lua 代码虽然执行了，但会无法更新 cocos2d-x 内部状态。

下面是 GameInterface_doBilling() 方法的主要代码：

``` java
public static void GameInterface_doBilling(final String billingIndex,
    final boolean useSms,
    final boolean isRepeated,
    final int luaFunctionId) {
  
  LuaJavaBridge.retainLuaFunction(luaFunctionId);

  context.runOnUiThread(new Runnable() {
    @Override
    public void run() {
      GameInterface.doBilling(useSms, isRepeated, billingIndex, new BillingCallback() {

        ...

        @Override
        public void onBillingSuccess() {
          context.runOnGLThread(new Runnable() {
            @Override
            public void run() {
              LuaJavaBridge.callLuaFunctionWithString(luaFunctionId, "success");
              LuaJavaBridge.releaseLuaFunction(luaFunctionId);
            }
          });
        }

        ...
      });
    }
  });
}
```

~

方法中，构造了一个 Runnable 对象，用来包装需要执行的 Java 代码。这个 Runnable 对象被指定运行在 UI 线程上。这样当调用 GameInterface.doBilling() 方法时就可以正确显示出支付界面。

当用户支付成功后，GameInterface.doBilling() 会调用 BillingCallback.onBillingSuccess() 方法。这个方法里构造了另一个 Runnable 对象，包装了调用 Lua function 的代码。

看上去代码不少，实际上就是在两个线程间互相切换。确保 Lua function 跑在 GL 线程，Java 代码跑在 UI 线程。

~


### Lua function 的引用计数器 ###

Lua 虚拟机具有自动垃圾回收机制。Lua function 既然是值，那么在没有被使用时自然会被回收掉。所以 luaj 提供了 retainLuaFunction() 和 releaseLuaFunction() 两个函数用于增减 Lua function 的引用计数。

将一个 Lua function 以引用 ID 的形式传入 Java 后，如果需要异步调用该 Lua function，那么就必须用 retainLuaFunction() 增加计数器，避免 Lua function 被 Lua 虚拟机回收。而在 Lua function 不再使用后，也应该调用 releaseLuaFunction() 减少计数器。

如果了解 cocos2d-x 中 CCObject 的 autorelease 机制，那么对引用计数应该很熟悉，两者是完全相同的实现机制。

~


## 连接第三方 SDK 和 cocos2d-x 的中间层 ##

虽然 luaj 可以让开发者从 Lua 中直接调用 Java 代码。但大部分第三方 SDK 在初始化时都需要指定当前应用程序的 Activity 对象，并且还要切换不同线程，所以对于大多数第三方 SDK，我们仍然要写一个中间层用于 Lua 和 Java 的交互。

与使用 JNI 做中间层相比，配合 luja 的中间层是使用 Java 来编写的，不但更简单明了，而且处理线程切换也非常简单。

~

要实现一个中间层，只有两个步骤：

-   实现初始化方法，用于将应用程序的 Activity 对象传入 SDK；
-   实现供 luaj 调用的 Java 接口

下面就是我们项目中实际用到的[“中国移动游戏基地和短信支付 SDK”中间层源代码](https://gist.github.com/4430966)：

``` java “中国移动游戏基地和短信支付 SDK”中间层

package com.qeeplay.frameworks;

import org.cocos2dx.lib.Cocos2dxActivity;

import android.content.Intent;
import android.util.Log;
import cn.emagsoftware.gamebilling.api.GameInterface;
import cn.emagsoftware.gamebilling.api.GameInterface.BillingCallback;
import cn.emagsoftware.gamebilling.api.GameInterface.GameExitCallback;
import cn.emagsoftware.gamecommunity.api.GameCommunity;

public class ChinaMobile_SDK
{
  private static Cocos2dxActivity context;
  
  public static void setContext(Cocos2dxActivity context_) {
    context = context_;
  }
  
  public static void GameCommunity_initializeSDK(final String appName,
      final String key,
      final String secret,
      final String appId) {
    context.runOnUiThread(new Runnable() {
      @Override
      public void run() {
        GameCommunity.initializeSDK(context,
            appName,
            key,
            secret,
            appId,
            "com.qeeplay.games.killfruitcn");
      }
    });
  }
  
  public static void GameCommunity_exit() {
    context.runOnUiThread(new Runnable() {
      @Override
      public void run() {
        GameCommunity.exit();
      }
    });
  }
  
  public static void GameCommunity_launchGameCommunity() {
    context.runOnUiThread(new Runnable() {
      @Override
      public void run() {
        GameCommunity.launchGameCommunity(context);
      }
    });
  }

  public static void GameCommunity_launchGameRecommend() {
    context.runOnUiThread(new Runnable() {
      @Override
      public void run() {
        GameCommunity.launchGameRecommend(context);
      }
    });
  }
  
  public static void GameCommunity_commitScoreWithRank(
      final String leaderboardId,
      final int scoreValue) {
    context.runOnUiThread(new Runnable() {
      @Override
      public void run() {
        GameCommunity.commitScoreWithRank(context, leaderboardId, scoreValue);
      }
    });
  }
  
  // ----------------------------------------

  public static void GameInterface_initializeApp(final String appName,
      final String cpName,
      final String tel) {
    GameInterface.initializeApp(context, appName, cpName, tel);
  }
  
  public static void GameInterface_doBilling(final String billingIndex,
      final boolean useSms,
      final boolean isRepeated,
      final int luaFunctionId) {
    LuaJavaBridge.retainLuaFunction(luaFunctionId);
    context.runOnUiThread(new Runnable() {
      @Override
      public void run() {
        GameInterface.doBilling(useSms, isRepeated, billingIndex, new BillingCallback() {
          @Override
          public void onUserOperCancel() {
            context.runOnGLThread(new Runnable() {
              @Override
              public void run() {
                LuaJavaBridge.callLuaFunctionWithString(luaFunctionId, "cancel");
                LuaJavaBridge.releaseLuaFunction(luaFunctionId);
              }
            });
          }
          
          @Override
          public void onBillingSuccess() {
            context.runOnGLThread(new Runnable() {
              @Override
              public void run() {
                LuaJavaBridge.callLuaFunctionWithString(luaFunctionId, "success");
                LuaJavaBridge.releaseLuaFunction(luaFunctionId);
              }
            });
          }
          
          @Override
          public void onBillingFail() {
            context.runOnGLThread(new Runnable() {
              @Override
              public void run() {
                LuaJavaBridge.callLuaFunctionWithString(luaFunctionId, "fail");
                LuaJavaBridge.releaseLuaFunction(luaFunctionId);
              }
            });
          }
        });
      }
    });
  }
  
  public static void GameInterface_setActivateFlag(final String billingIndex,
      boolean flag) {
    GameInterface.setActivateFlag(billingIndex, flag);
  }
  
  public static boolean GameInterface_getActivateFlag(final String billingIndex) {
    return GameInterface.getActivateFlag(billingIndex);
  }
  
  public static boolean GameInterface_isMusicEnabled() {
    return GameInterface.isMusicEnabled();
  }
  
  public static void GameInterface_viewMoreGames() {
    context.runOnUiThread(new Runnable() {
      @Override
      public void run() {
        GameInterface.viewMoreGames(context);
      }
    });
  }
  
  public static void GameInterface_exit(final int luaFunctionId) {
    LuaJavaBridge.retainLuaFunction(luaFunctionId);
    context.runOnUiThread(new Runnable() {
      @Override
      public void run() {
        GameInterface.exit(new GameExitCallback() {
          @Override
          public void onConfirmExit() {
            context.runOnGLThread(new Runnable() {
              @Override
              public void run() {
                LuaJavaBridge.callLuaFunctionWithString(luaFunctionId, "exit");
                LuaJavaBridge.releaseLuaFunction(luaFunctionId);
                context.runOnUiThread(new Runnable() {
                  @Override
                  public void run() {
                    System.exit(0);
                  }
                });
              }
            });
          }
          
          @Override
          public void onCancelExit() {
            context.runOnGLThread(new Runnable() {
              @Override
              public void run() {
                LuaJavaBridge.callLuaFunctionWithString(luaFunctionId, "cancel");
                LuaJavaBridge.releaseLuaFunction(luaFunctionId);
              }
            });
          }
        });
      }
    });
  }

}

```

~

SDK 需要传入当前程序的 Activity，所以需要在游戏的 onCreate() 中调用 setContext() ：

``` java
public class mygame extends Cocos2dxActivity {

  protected void onCreate(Bundle savedInstanceState) {
    ChinaMobile_SDK.setContext(this); // init sdk
    super.onCreate(savedInstanceState);
  }
  
  ...
  
}
```

~

做好一切准备工作后，在游戏的 Lua 代码里访问 SDK 功能就很简单了：

``` lua

local luaj = require("luaj")

local className = "com/qeeplay/frameworks/ChinaMobile_SDK"

-- 初始化 SDK
local args = {
  CHINA_MOBILE_SP_APP_NAME,
  CHINA_MOBILE_SP_CP_NAME,
  CHINA_MOBILE_SP_TEL
}
luaj.callStaticMethod(className, "GameInterface_initializeApp", args)

-- 支付
local function callback(result)
  if result == "success" then
    -- 支付成功
  end
end

local args = {
  billingIndex,
  true,
  true,
  callback
}
luaj.callStaticMethod(className, "GameInterface_doBilling", args)

-- 显示游戏基地界面
luaj.callStaticMethod(className, "GameCommunity_launchGameCommunity")

-- 提交玩家的游戏成绩
local args = {
  "0",            -- 排行榜Id
  newBestScores,  -- 新的最佳成绩
}
local sig = "V(SI)"
luaj.callStaticMethod(className, "GameCommunity_commitScoreWithRank", args, sig)

```

~


## 安装 luaj ##

luaj 分为三个部分：

-   LuaJavaBridge.java, com\_qeeplay\_frameworks\_LuaJavaBridge.h/.cpp - 供 Java 端使用的工具类。
-   LuaJavaBridge.h/.cpp - 供 Lua 端使用的工具类。
-   luaj.lua - LuaJavaBridge 的 Lua 包装，提供更简单和灵活的接口。

步骤：

-   将 LuaJavaBridge.java 添加到 Android 项目中；
-   修改 proj.android/jni/Android.mk：

``` makefile

LOCAL_SRC_FILES := ...
    luaj/jni/com_qeeplay_frameworks_LuaJavaBridge.cpp \
    luaj/luabinding/LuaJavaBridge.cpp
    
LOCAL_C_INCLUDES := ...
    luaj

```

-   修改 AppDelegate.cpp，加入以下代码：

```
#if (CC_TARGET_PLATFORM == CC_PLATFORM_ANDROID)
#include "LuaJavaBridge.h"
#endif

bool AppDelegate::applicationDidFinishLaunching()
{
  
  ...
  
  CCLuaEngine* pEngine = CCLuaEngine::defaultEngine();
  CCScriptEngineManager::sharedManager()->setScriptEngine(pEngine);
  
  LuaJavaBridge_luabinding_open(pEngine->getLuaState());
  
  ...
}
```

-   修改proj.android/jni/hellocpp/main.cpp，加入以下代码：

``` c++
jint JNI_OnLoad(JavaVM *vm, void *reserved)
{
  
  ...
  
  LuaJavaBridge_setJavaVM(vm);
  
  return JNI_VERSION_1_4;
}

```

~


## luaj 方法参考 ##

-   ```[Lua] luaj.callStaticMethod(className, methodName, args, methodSig)```

    调用指定的 Java class static method，允许传入 int/float/boolean/string/function 五种类型的参数。


-   ```[Lua] luaj.callStaticMethodWithArray(className, methodName, array)```

    将一个 Lua table 数组转换为 Java 的 Vector 集合，并传入被调用的函数。适合批量传递数据给 Java。数组中值的支持 int/float/boolean/string 四种类型。

    callStaticMethodWithArray() 要求 Java 方法必须定义为 GetArray() 的形式（方法名随意）：只有一个 Vector 类型的参数，并返回 int 值。

    用法：

``` lua
local array = {"stringValue", 1, 3.14, true}
local ok, ret = luaj.callStaticMethodWithArray(className, "GetArray", array)
```

``` java
public static int GetArray(Vector vector) {
  for (Enumeration e = hash.keys(); e.hasMoreElements();)
  {
    String key = (String)e.nextElement();
    Log.d("testWithHash", "  [" + key + "] = " + hash.get(key).toString());
  }
  return 0;
}

```

~

-   ```[Java] LuaJavaBridge.callLuaFunctionWithString(int luaFunctionId, String value)```

    调用引用 ID 指向的 Lua function，并传入一个字符串作为参数。


-   ```[Java] LuaJavaBridge.callLuaGlobalFunctionWithString(int luaFunctionId, String value)```

    调用指定名字的 Lua 全局函数，并传入一个字符串作为参数。
    

-   ```[Java] LuaJavaBridge.retainLuaFunction(int luaFunctionId)```

    增加引用 ID 的计数，确保 Lua function 不会被 Lua 虚拟机自动回收。


-   ```[Java] LuaJavaBridge.releaseLuaFunction(int luaFunctionId)```

    减少引用 ID 的计数，当计数等于 0 时，引用 ID 指向的 Lua function 将被回收。

~


## 未来改进 ##

因为我们自己的项目暂时还没有更复杂的需求，所以 luaj 目前的实现很简单。但要在这个基础上进行完善是很容易的事情，luaj 已经解决了几个关键性问题。

未来计划会增加的主要特性就是支持更多的类型，例如将一个以字符串做键的 Lua table 以 Java Map 集合的形式传递给 Java。同样，从 Java 调用 Lua 函数时，也应该支持多个参数，以及更多的参数类型。

至于将 Java 对象传入 Lua，并在 Lua 中调用 Java 对象的方法，目前没这个打算。因为 luaj 的主要目的是为 cocos2d-x 游戏服务，而 cocos2d-x 的多线程模式要求 Lua 和 Java 代码必须在不同的线程里运行。如果在 Lua 中调用 Java 对象方法将面对许多复杂的问题。与其花大量时间去解决这个问题（还不一定能保证最后简单易用），不如简单写一个中间层。

最后，luaj 已经被集成到了 quick-cocos2d-x 这个基于 cocos2d-x 的快速游戏开发引擎中。quick-cocos2d-x 让开发者可以使用 Lua 语言开发一个完整的大型商业游戏，同时又保持 cocos2d-x 的高性能、开放性、可扩展能力。

最后的最后，惯例为广大程序猿送上福利美图一张 :-)

![](/upload/2013-01/2013-01-01.jpg)


