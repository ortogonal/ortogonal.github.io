---
layout: post
title: "Using ccache together with qmake and QtCreator"
description: "How to integrate ccache with qmake and QtCreator? Since version 5.9.2 there is a really simple way."
tags: [qt, qmake, qtcreator]
modified: 2019-01-30
image:
  path: /images/abstract-8.jpg
#  feature: abstract-8.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
---

`ccache` is a simple tool that speeds up your builds/rebuilds. It builds up a cache that is used when building and rebuilding. This cache can make a huge performance impact on your build time. On there web page it says.

>  ccache is a compiler cache. It [speeds up recompilation](https://ccache.samba.org/performance.html) by caching previous compilations and detecting when the same compilation is being done again. Supported languages are C, C++, Objective-C and Objective-C++.

# Integrating `ccache` into your build
I've been searching the web for a simple solution to add ccache into a `qmake`-based project with out any real luck. I found some solutions were people have done some minor hacks like:
```cpp
# TLDR? No there are better ways...
QMAKE_CXX = ccache $$QMAKE_CXX
```
Put that in your `.pro`-file and `ccache` will sort of work. But since version 5.9.2 of Qt there is a much simpler solution. Tor Arne Vestbø made a nice [commit](https://github.com/qt/qtbase/commit/d64940891dffcb951f4b76426490cbc94fb4aba7) that simplifies this alot.

To use this feature you either add `load(ccache)` into your `.pro`-file or add `CONFIG+=ccache` to your `qmake` command line.

How much performance gain your project gets is hard to say - so test and see if it will improve your build.

# `ccache` installation
Unfortunately the `ccache` feature in qmake doesn't check whether you have `ccache` installed or not. This is something that is assumed. To install `ccache` on a Ubuntu based Linux machine run `sudo apt install ccache`. To solve it on other platforms/distributions I would recommend just simply Google it!   

# `ccache` configuration
One can configure `ccache` in many different ways. The default config (on Ubuntu 18.04) seems to be a maximum cache size of 5 GB. This can easily be changed. More information can be found [here](https://ccache.samba.org/manual/latest.html).

# Build Qt with `ccache`
The commit I mentioned above also add support for `ccache` into the Qt5 build. This is something that will speed up your build and also fill your chache :).
