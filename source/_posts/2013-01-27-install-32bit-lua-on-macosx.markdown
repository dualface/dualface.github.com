---
layout: post
title: "在 Mac OS X 上安装 Lua 32bit 环境"
date: 2013-01-27 11:42
comments: true
categories: lua
---
因为我们公司使用 [quick-cocos2d-x](https://github.com/dualface/quick-cocos2d-x)（一个基于 cocos2d-x 的 Lua 游戏引擎）开发，所以在开发环境里必须配置 Lua，并且需要和移动设备上的 32bit Lua 匹配。

默认情况下，OS X 上编译 Lua 会编译成 64bit，所以需要做点手脚。此外还需要修改 LuaRocks，不然用 Lua 是编译成 32bit 了，可用 LuaRocks 安装的扩展还是 64bit。

<!--more-->

## 编译 32bit Lua

下载源代码（我用的 Lua 5.1.5）后，修改文件 src/Makefile，找到：

``` makefile
macosx:
    $(MAKE) all MYCFLAGS=-DLUA_USE_LINUX MYLIBS="-lreadline"
```

改为：

``` makefile
macosx:
    $(MAKE) all MYCFLAGS="-arch i386 -DLUA_USE_LINUX" MYLIBS="-arch i386 -lreadline"
```

make 后，file src/lua 可以看到已经是 32bit 执行文件了。

``` bash
$ make
$ file src/lua
src/lua: Mach-O executable i386
```

<br />

## 让 LuaRocks 安装 32bit 扩展

首先安装 LuaRocks，然后修改文件 /usr/local/share/lua/5.1/luarocks/cfg.lua，找到：

``` lua
if detected.macosx then
   ...
   defaults.variables.LIBFLAG = "-arch i386 -bundle -undefined dynamic_lookup -all_load"
   ...
end
```

改为：

``` lua
if detected.macosx then
   ...
   defaults.variables.CFLAGS = "-arch i386"
   defaults.variables.LIBFLAG = "-arch i386 -bundle -undefined dynamic_lookup -all_load"
   ...
end
```

增加 CFLAGS 标志，并在 LIBFLAG 中添加 "-arch i386"。修改完成后，用 LuaRocks 重新安装需要的扩展。

``` bash
$ sudo luarocks install luafilesystem
...
luafilesystem 1.6.2-1 is now built and installed in /usr/local/

$ file /usr/local/lib/lua/5.1/lfs.so
/usr/local/lib/lua/5.1/lfs.so: Mach-O bundle i386
```

\- 完 \-

PS: 找了半天没找到含而不露的福利图 -3-
