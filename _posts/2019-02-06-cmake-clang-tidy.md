---
layout: post
title: "Integrate Clang-Tidy into CMake"
description: "Clang-Tidy is a really great tool that does static code analysis on your code. To be honest I've become a much better C++ developer thanks to Clang-Tidy. So how can this be integrated into CMake."
tags: [cmake, clang-tidy, Better C++]
modified: 2019-02-06
image:
  path: /images/abstract-8.jpg
#  feature: abstract-8.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
---

Clang-Tidy is a really great tool that does static code analysis on your code. To be honest I've become a much better C++ developer thanks to Clang-Tidy.

My favorite IDE, QtCreator, has something called Clang-Code-Model which is a plugins that runs in the background and in real-time displays things that can be changed. QtCreator also has a tool which can run Clang-Tidy on your project to list potential problems. All this is very nice and makes both you and your code better. But there is one thing missing - integrate into your build.

So I thought I should write a bit on how you can implement Clang-Tidy into your CMake-based project in a really simple way. The goal is that we run Clang-Tidy on the files we compile and not the complete project. The reason for this is speed and also to "force" all team-members to use it.
## CMake and Clang-Tidy
Since version 3.7.2 of CMake there is a really simple solution to this. Just set `CMAKE_CXX_CLANG_TIDY` (there is `CMAKE_C_LANG_TIDY` for your C code as well) and your done! At least almost.
You can set `CMAKE_CXX_CLANG_TIDY` either in your `CMakeList.txt` or you can do it from the command line.
```
set(CMAKE_CXX_CLANG_TIDY "clang-tidy;-checks=*")
```
or in `CMakeLists.txt`
```
set(CMAKE_CXX_CLANG_TIDY "clang-tidy;-checks=*")
```
These two example above will run Clang-Tidy with all checks enabled.
### Passing arguments to Clang-Tidy
The string you pass the `CMAKE_CXX_CLANG_TIDY` is your different arguments to Clang-Tidy. So the example above passes `-checks=*` to Clang-Tidy.

### Getting checks on headers
By default Clang-Tidy will not check your header files for problems. But you might be interested in running Clang-Tidy on your headers as well. This is done with the argument `-header-filter=.`. For example:
```cpp
set(CMAKE_CXX_CLANG_TIDY 
  clang-tidy;
  -header-filter=.;
  -checks=*;)
```
The above snippet from `CMakeLists.txt` will make sure to also check header in the `.`-folder, the project folder. You might need to change this depending on your project setup.
### Treating warning as errors
A really nice feature is that you can treat warnings as errors if you like. This can be enabled at any time or maybe just on your CI-build.

The idea is that you configure a number of checks using `-checks=...` and you configure which of those shall be treated as errors.

Lets say we enable two (randomly selected checks), `-checks=bugprone-*,cppcoreguidelines-avoid-goto`. Now we set which we would like to treat as errors.
If we set `-warnings-as-errors=*` both of the two tests above will generate errors. If you set `warnings-as-errors=cppcoreguidelines-avoid-goto` only the test `cppcoreguidelines-avoid-goto`  will be treated as an error.

Remember that by setting `-warnings-as-errors=...`you don't enable any test, this must be done by using `-checks=`.

Below is a example how `-warnings-as-errors` looks in CMake.
```cpp
set(CMAKE_CXX_CLANG_TIDY
  clang-tidy;
  -header-filter=.;
  -checks=*;
  -warnings-as-errors=*;)
```
## Example repo
I've made a simple example, with some problems that Clang-Tidy finds. You can find it at my [Github repo](https://github.com/ortogonal/cmake-cling-tidy-example).
