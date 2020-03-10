---
layout: post
title: "Bidirectional layouts in QML"
description: "Implement bidrectional layout (or layout mirroring) in QML"
tags: [qt, qml]
category: qt
modified: 2019-11-09
image:
  path: /images/abstract-8.jpg
#  feature: abstract-8.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
---

Handling multiple languages in Qt and QML is simple thanks to `tr()` and `qsTr()`, but what about handling bidirectional layouts? Well - develop your QML in a good way and it's simple, two lines of code if you are lucky. But in most cases some more - but just some.

In this post I would like to share some of the ways to handle this in QML and some experience I got doing this.

## LTR vs. RTL
Having read from left to right my whole life it was pretty strange seeing a user interface switch to RTL (right-to-left). More or less everything is mirrored. If your application "flows" from left to right this flow shall be changed.  But not everything is mirrored in a user interface. Googles Material Design has a really great guide how to handle layout mirroring that you can find [here]([https://material.io/design/usability/bidirectionality.html#mirroring-layout](https://material.io/design/usability/bidirectionality.html#mirroring-layout)). This is the best guide I found and I guess that Google knows what they are talking about.

## Steps for success
First of all - Qt and QML handles this for you if you develop your application in a smart way! **Don't reinvent the wheel one more time!**

### Setting the layout from C++ or QML
First of all before we start to look at how to mirror a layout we need a way of telling the application that it shall be mirrored. This can be done either in C++ or QML.

The `QGuiApplication` has a property that is called [`layoutDirection()`]([https://doc.qt.io/qt-5/qguiapplication.html#layoutDirection-prop](https://doc.qt.io/qt-5/qguiapplication.html#layoutDirection-prop)) that returns a `Qt::LayoutDirection`. `Qt::LayoutDirection` can be either `Qt::LeftToRight` or `QtRightToLeft`, there is also a `Qt::LayoutDirectionAuto` which for now is out of scope.

This property can be set from either C++ or QML when you would like to mirror your layout. It is also very useful to use in your QML code to know when to switch to Right-To-Left layout.


```javascript
Qt.application.layoutDirection === Qt.RightToLeft
```
*Example: How to know if you are using Right-To-Left layout or not.*

### QML and LayoutMirroring
In QML there is an attached property called `LayoutMirroring`. An attached property is a property that can be added to any other QML item to give it more properties/features. Attached properties is probably worth its own post in the future.

What `LayoutMirroring` does is simply to mirror the layout (hence the name). If it is enabled it will transform your `anchors.left: foo.left` to `anchors.right` and vice versa.

`LayoutMirroring` holds two properties, `enabled` and `childrenInherit`. Enable is simple to understand, on or off. The property `childrenInherit` is a boolean that will set `LayoutMirroring` on **all** children components. This means that if you set this property on your QML root node/item it will propagate this feature on to all other children in the object hierarchy. So two simple lines can make a huge difference!
```cpp
Window {
  width: 200
  height: 200

  LayoutMirroring.enabled: Qt.application.layoutDirection === Qt.RightToLeft
  LayoutMirroring.childrenInherit: true
}
```
*Example: Setting `LayoutMirroring` in QML.*

### Anchoring is your friend
To get the `LayoutMirroring` to work it requires you to use anchoring when positioning your items in QML, otherwise the layout mirroring has no effect.
```cpp
Item {
  // Big no no - use anchors
  Recthangle {
    x: 10; y: 10
    width: 20; height: 20
    color: "red"
  }

  // Much better
  Rectangle {
    anchors: {
      top: parent.top
      left: parent.left
      topMargin: 10
      leftMargin: 10
    }
    width:  20; height:  20
    color: "red"
  }
}
```
This can be a big task to switch from not using anchors to using it. So make sure to use anchors from the start.


### Disable layout mirroring on some components
As you can read in Google's guidelines some components shall not be mirrored. One such thing is a progressbar that visualize the passage of time. This means that we must be able to disable layout mirroring for some components.

I think it's best to enable `LayoutMirroring` in your root QML object of your application instead of setting it for each single component/item. It is really simple to disable it, just set `LayoutMirroring.enabled` to false on that specific component you don't want to mirror. You might need to set the `inheritChildren` property also if your components holds children.

### Handle icons and images
`LayoutMirroring` only mirrors the layout, not icons/images this is something that needs to be handled by you. The `Image` item in QML can easily be mirrored using the property `mirror`. Set that to `true` and the icon will be mirrored.

This, like anchoring changes, can be a huge task to do, so instead of setting this mirror property on every single place your use `Image` divide them into simple QML components. For example, take a UI similar to the iPhone settings menu.

![Example]({{ site.url }}/images/45877.jpg)
{: .image-left}
*Image: Example from my iPhone (also some Swedish text, but you get the sence for it, right?*

The QML (pseudo code) for this might look something like this:
```cpp
ListView {
    model: fooModel
    delegate: MyRow {
        width: ListView.view.width
        Image {
            id: icon
            anchors: {
                left: parent.left
                leftMargin: 40
                verticalCenter: parent.verticalCenter
            }
            source: iconSource
        }
        Text {
            anchors: {
                left: icon.left
                leftMargin: 40
                verticalCenter: parent.verticalCenter
            }
            text: itemText
        }
        Image {
            anchors: {
                right: parent.right
                leftMargin: 40
                verticalCenter: parent.verticalCenter
            }
            source: "arrow.png"
        }
    }
}
```
The problem with this code is the last `Image`, that renders the small arrow icons to the right of each row. When using `LayoutMirroring` this arrow icon will be on the left side instead of the right, **but** the icon it self will not be mirrored. This can be fixed by setting the `mirror` property:

```cpp
Image {
    mirror: Qt.application.layoutDirection === Qt.RightToLeft
    anchors: {
        right: parent.right
        leftMargin: 40
        verticalCenter: parent.verticalCenter
    }
    source: "arrow.png"
}
```
This works fine, but your code will have a bunch of these `Qt.application.layoutDirection === Qt.RightToLeft` here and there to mirror images/icons. One thing you can do to make this simpler is to create a `MirroredImage`- component
```cpp
// MirrorImage.qml
Image {
    mirror: Qt.application.layoutDirection === Qt.RightToLeft
}
```

Then it is simple to exchange your `Image`-items to `MirroredImage`-items when you would like to have an image/icons mirrored. It is just one line of code you save, but it makes to code much more readable!

## Conclusion
Thanks to Qt and QML (as always) this is a quite simple task, just stick to using `anchors`!
