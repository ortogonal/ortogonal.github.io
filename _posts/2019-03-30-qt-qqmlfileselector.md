---
layout: post
title: "Let QQmlFileSelecor select which QML file to use"
description: "One problem I've been facing now and then when developing QML is the need for selecting different QML files depending on hardware, operating system, customer brand etc. QML has a pretty unknown feature for this called QQmlFileSelecor"
tags: [Qt, qt, QML, qml]
modified: 2019-03-30
image:
  path: /images/abstract-8.jpg
#  feature: abstract-8.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
---

One problem I've been facing now and then when developing QML is the need for selecting different QML files depending on hardware, operating system, customer brand etc. QML has a pretty unknown feature for this called [`QQmlFileSelecor`]([https://doc.qt.io/qt-5/qqmlfileselector.html](https://doc.qt.io/qt-5/qqmlfileselector.html))

There are many different use-cases for selecting different QML-file in your application. It can be that you are developing an application that has different UI depending on the screen size (maybe one 3" display and one 7" display in an embedded system). It can also be that you are developing a product that runs under multiple brands but still have more or less the same QML. Or it can simply be the case for debug/release or Window/Linux.

## Example
[`QQmlFileSelecor`]([https://doc.qt.io/qt-5/qqmlfileselector.html](https://doc.qt.io/qt-5/qqmlfileselector.html)) lets you do this in a simple way. What `QQmlFileSelector` does is to configure a number of selectors which QML will use when searching for a file.  Let us look at an example.
```cpp
QQmlApplicationEngine engine;

QStringList selectors;
selectors << "desktop";

// Our QQmlApplicationEngine will take owner-ship of fs.
QQmlFileSelector *fs = new QQmlFileSelector(&engine);
fs->setExtraSelectors(selectors);
engine.load(QUrl(QStringLiteral("qrc:/main.qml")));
```
The above code is pretty standard, the only difference is:
```cpp
QQmlFileSelector *fs = new QQmlFileSelector(&engine);
fs->setExtraSelectors(selectors);
```
What we are doing here is actually creating and adding a `QQmlFileSelector` to the QML-engine with an added extra selector called `desktop`. When the QML-engine starts to search for `main.qml` it will look for files in:
```
./+desktop/main.qml
./main.qml
```
So now it's easy do setup a project were you have one (or more) QML-files that are desktop specific just by adding them in the `+desktop`-folder. Off course you need some code in C++ that creates the correct selector.
```cpp
if (runningOnDesktop()) {
	// Add your desktop selector
}
```
You can download the full example at:
## Default selectors
To make life even simpler the `QQmlEngine` has a default selector that is always available. The default selector adds your locale language and operating system information. On my Linux KDE Neon computer my default selectors are:
```
("en_US", "unix", "linux", "neon")
```
So if I would like to add a specific QML-file for my application that is only used when running Linux I can just create a folder called `+linux` and place my content there.

## Nested selectors
`QQmlFileSelector` lets you use nested selectors as well. Let's go back to the example above. We have added a selector called `desktop`, but we also have the default selectors available. So our list of selectors looks like:
```
("desktop", "en_US", "unix", "linux", "neon")
```
*To get your list of selectors look at [`QQmlFileSelector::selector()::allSelectors()`]([https://doc.qt.io/qt-5/qfileselector.html#allSelectors](https://doc.qt.io/qt-5/qfileselector.html#allSelectors))*

If we then create a folder structure that looks like:
```
+desktop/+linux/main.qml
+desktop/main.qml
main.qml
```
We will have one specific `main.qml` when we run on desktop and Linux (` +desktop/+linux/main.qml`), one when we run on Linux (and not desktop) (`+desktop/main.qml`) and finally the `main.qml` for all other cases.

## QFileSelector
Qt also has a `QFileSelector`-class that can by used outside the QML-engine to get the same feature. It can be used to load different images depending on selectors etc. But that is another post!
