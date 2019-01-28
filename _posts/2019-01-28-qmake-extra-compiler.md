---
layout: post
title: "QMAKE_EXTRA_COMPILERS in qmake"
description: "Let's look at how we can use QMAKE_EXTRA_COMPILERS to execute things before our normal compilation."
tags: [qt, c++, qmake]
modified: 2019-01-28
image:
  path: /images/abstract-7.jpg
  feature: abstract-7.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
---

When reading the documentation for `qmake` one finds something called `QMAKE_EXTRA_COMPILERS`. The documentation is clear, but not really an everyday use-case. So I thought I would try to explain `QMAKE_EXTRA_COMPILERS` with a "real" use-case from a project I'm currently working on for one of Combitechs many customers.

# Background
In this project we generate code based on the content of a number of XML-files. Maybe one day I should write a bit about why we are generating code and how we wrote the code-generator as well. But for now the only thing we need to know is that there is a list of XML-files that we use as input to the code-generator and out comes a number of header-files.

When I proposed the code-generator for the rest of the project I promised them that this should be simple and easy to use for the rest. I might have said something like:

> Just a parameter with the XML-files in the pro-file

So integrating the code-generation into our `qmake`-based build was of highest priority. And thanks to `QMAKE_EXTRA_COMPILERS` it was fairly simple.

# What does QMAKE_EXTRA_COMPILERS do
You can read the full documentation of QMAKE_EXTRA_COMPILERS at [qt.io](http://doc.qt.io/qt-5/qmake-advanced-usage.html#adding-compilers). But in short:
- you setup an executable that qmake will execute. 
- you provide the executable with input argument. You do not have to do, but would be strange not to.
- you add, if needed the output or data to some other `qmake`-parameter.

The example on qt.io is based around `moc` and how to setup your own `moc`-solution. But in this example we are going to pass a list of XML-files in to this.

# Setting it up for code-generation
The goal was to keep the implementation in the main project file as simple as possible.
```cpp 
CG_INPUT += \
    fileA.xml \
    fileB.xml
include(code-generator.pri)
```
The above code will just create a parameter call `CG_INPUT` that, in this case, holds fileA.xml and fileB.xml. The next thing we do is to load `code-generator.pri`.
```cpp
cg.name = "Internal code generator"
cg.input = CG_INPUT
cg.output = $$OUT_PWD/generated/${QMAKE_FILE_BASE}$${first(QMAKE_EXT_H)}
cg.variable_out = HEADERS
cg.commands = \
    $$OUT_PWD/../code-generator/bin/cg -s -o $$OUT_PWD/generated -r ${QMAKE_FILE_IN}
QMAKE_EXTRA_COMPILERS += cg

DISTFILES += PROTOCOLS
```
Let's look what the code above does. `cg.name` just sets a name on the extra compiler we are adding. `cg.input = CG_INPUT` tells the extra compiler what its input are, in this case our XML-files (fileA.xml and fileB.xml). 

`cg.output` list the output files. This is where it starts to get a bit strange, to be honest. [`QMAKE_FILE_BASE`](https://wiki.qt.io/Undocumented_QMake#Custom_tools) is one of de "undocumented" things in `qmake`. What it will do is actually take the filename of, each and every, file from `cg.input` and remove the file-ending (`.xml`). After that we add the header-file ending. So our outputs, in this example, will be fileA.h and fileB.h.

`cg.variable_out` is very useful. You use that to tell which parameter to append the output files to. In this example we append them to the `HEADERS`-list. This will make sure `qmake` knows about them.
`cg.command` is the command you would like to execute. I've used `${QMAKE_FILE_IN}` as input to our code generator. `${QMAKE_FILE_IN}` will be fileA.xml resp. fileB.xml. Because we have two XML-files this application will be called twice.

The last, and most important thing, is to append `cg`to the list of extra compilers. This is done by adding `QMAKE_EXTRA_COMPILERS += cg`. The name `cg` is short for code-generation but this variable could have any name, not just `cg`.
# Running it in QtCreator/qmake
This setup makes the use and integration of a proprietary code-generator really simple in QtCreator and qmake. It requires a really small amount of code in the project files to pull this simple hack off.

One thing that took a while to understand is that qmake will execute the `cg.command` once for every file in `CG_INPUT`, but only if the output header-file is used somewhere in the code.
Offcourse it holds dependencies "back" to the XML-file. If the XML-file content is change the code-generator will be executed on next compilation.
