---
layout: post
title: "tolua++ 实现分析"
date: 2012-08-25 11:05
comments: true
categories: lua
---

tolua++ 是一个将 C/C++ 的函数和对象导出给 Lua 脚本使用的工具。

使用这个工具的基本步骤：

-   将要导出的 C/C++ 函数和对象定义写入 .pkg 文件；
-   运行 tolua++ 工具，将 .pkg 文件编译为目标 .cpp 文件；
-   将目标 .cpp 文件加入项目，在启用 Lua 虚拟机后调用目标文件中的 open() 函数注册导出的内容。

<!--more-->

~

## tolua++ 工具生成的目标文件 ##

每个目标文件都是从一系列 .pkg 文件编译而来，主要完成下列功能：

-   定义所有从 C/C++ 导出的类型、函数和对象；
-   注册所有类型及其方法。

~

### 定义要导出的方法

不管是 C 函数还是 C++ 对象的方法，都一律导出为静态函数。

C 函数的导出形式如下：

``` lua
static int tolua_math2dx_luabinding_dist00(lua_State* tolua_S)
{
    tolua_Error tolua_err;
    if (!tolua_isnumber(tolua_S,1,0,&tolua_err) ||
        !tolua_isnumber(tolua_S,2,0,&tolua_err) ||
        !tolua_isnumber(tolua_S,3,0,&tolua_err) ||
        !tolua_isnumber(tolua_S,4,0,&tolua_err) ||
        !tolua_isnoobj(tolua_S,5,&tolua_err))
    {
        goto tolua_lerror;
    }
    else
    {
        float p1x = (float)tolua_tonumber(tolua_S,1,0);
        float p1y = (float)tolua_tonumber(tolua_S,2,0);
        float p2x = (float)tolua_tonumber(tolua_S,3,0);
        float p2y = (float)tolua_tonumber(tolua_S,4,0);
        {
            float tolua_ret = dist(p1x, p1y, p2x, p2y);
            tolua_pushnumber(tolua_S, tolua_ret);
        }
    }
    return 1;
tolua_lerror:
    tolua_error(tolua_S,"#ferror in function 'dist'.",&tolua_err);
    return 0;
}
```

由于这个导出模块的名字是 math2dx_luabinding，所以导出函数的前缀就是 tolua_math2dx_luabinding_ 。而导出函数 dist 的名字也添加了后缀 00 用于区别可能存在的函数重载。

导出函数 dist00() 的执行步骤：

1.  首先从 stack 提取函数的参数，并一一判断类型。如果类型不符，则输出错误信息并中断执行。
2.  将参数复制到临时变量中，然后调用目标函数 dist()，并将结果（如果 dist() 有返回值）push 到 stack。
3.  dist00() 最后返回 dist() 函数的返回值的个数。

~

对于 C++ 对象，方法则分为类静态方法和实例方法两种情况。

由于 tolua++ 导出的 C++ 类静态方法用“:”操作符调用：

    local request = CCHttpRequest:create()

因此导出函数里要求传入的第一个参数是 CCHttpRequest 模块：

``` lua
// CCHttpRequest:create()
static int tolua_cocos2dx_extension_network_CCHttpRequest_create00(lua_State* tolua_S)
{
    tolua_Error tolua_err;
    if (!tolua_isusertable(tolua_S,1,"CCHttpRequest",0,&tolua_err) ||
        (tolua_isvaluenil(tolua_S,2,&tolua_err) ||
            !toluafix_isfunction(tolua_S,2,"LUA_FUNCTION",0,&tolua_err)) ||
        !tolua_isstring(tolua_S,3,0,&tolua_err) ||
        !tolua_isnumber(tolua_S,4,1,&tolua_err) ||
        !tolua_isnoobj(tolua_S,5,&tolua_err))
    {
        goto tolua_lerror;
    }
    else
    {
        LUA_FUNCTION listener = toluafix_ref_function(tolua_S,2,0);
        const char* url = (const char*)tolua_tostring(tolua_S,3,0);
        CCHttpRequestMethod method = (CCHttpRequestMethod)tolua_tonumber(tolua_S,
                4,CCHttpRequestMethodGET);
        {
            CCHttpRequest* tolua_ret = CCHttpRequest::create(listener,url,method);
            tolua_pushusertype(tolua_S,(void*)tolua_ret,"CCHttpRequest");
        }
    }
    return 1;
tolua_lerror:
    tolua_error(tolua_S,"#ferror in function 'create'.",&tolua_err);
    return 0;
}
```

PS: 个人认为将 C++ 对象视为一个 Lua module 时，那么类静态方法的调用方式应该是 CCHttpRequest.create() 这样，以便和实例方法区别开。

实例方法的导出区别不大，仅仅是需要检查 stack 中的第一个值是否是对象实例：

``` lua
// CCHttpRequest:start()
static int tolua_cocos2dx_extension_network_CCHttpRequest_start00(lua_State* tolua_S)
{
    tolua_Error tolua_err;
    if (!tolua_isusertype(tolua_S,1,"CCHttpRequest",0,&tolua_err) ||
        !tolua_isboolean(tolua_S,2,1,&tolua_err) ||
        !tolua_isnoobj(tolua_S,3,&tolua_err))
    {
        goto tolua_lerror;
    }
    else
    {
        CCHttpRequest* self = (CCHttpRequest*)tolua_tousertype(tolua_S,1,0);
        if (!self)
        {
            tolua_error(tolua_S,"invalid 'self' in function 'start'", NULL);
        }
        bool isCached = (bool)tolua_toboolean(tolua_S,2,false);
        self->start(isCached);
    }
    return 0;
tolua_lerror:
    tolua_error(tolua_S,"#ferror in function 'start'.",&tolua_err);
    return 0;
}
```

定义完所有要导出的方法后，tolua++ 目标文件将定义所有的模块并注册上述导出的方法。

~

### 注册模块和方法

对于 C 函数，会添加到 Lua 的全局名字空间中，而每一个 C++ 对象，则会注册一个与类名相同的 table，并添加到全局名字空间。

C++ 对象对应的 table 添加后，会将导出的类静态方法和实例方法添加到这个 table。

~

## tolua++ 库

要想让目标文件正常工作，还需要依赖 tolua++ 库提供的功能。这个库提供下列功能：

-   push 各种值到 lua stack；
-   push 一个对象或结构到 lua stack;
-   从 lua stack 取出值或对象等；
-   检查 lua stack 中值的类型；
-   在 lua 值被回收时删除对象实例。

其实这篇文章应该还有不少内容，但我实在是懒得写了。。。原谅我这个程序猿吧 -_-#

因为下一篇文章才是我要说的重点 :)

~

-EOF-
