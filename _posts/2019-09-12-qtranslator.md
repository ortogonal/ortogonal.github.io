---
layout: post
title: "QTranslator - handle your translations"
description: "Implement your own translator in Qt and QML"
tags: [qt, c++]
modified: 2019-09-12
image:
  path: /images/abstract-8.jpg
#  feature: abstract-8.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
---
So - first of all! Qt has a really nice suite of tools for handling translations. I'm not writing this to say that use this instead of there suite. This is just an alternative that might solve some needs.

Handling translations in Qt is super super simple. In C++ code you use the `QObject::tr(...)`-function to both mark a string for translation and to translate the string it self (in QML you use `qsTr()`). So for example.
```cpp
auto button = new QPushButton(tr("My Button"), this);
```
To extract the strings that needs to be translated from your application you use the Qt tool lupdate. lupdate will scan source code for places were `QObject::tr(...)` is used and create an XML-file that can be imported into Qt Linguist. After everything is translated you export the translations into a binary format that is then used within you application.

# But what is happening under the hood with tr()
Maybe I will over simplify this a bit. What happens is that

- When the application is started the QCoreApplication will setup a QTranslator that loads the exported translations (depending on your application) from Qt Linguist/lconvert.
- When you call `QObject::tr(...)` it will call `QCoreApplication::translate(...)` which will use the QCoreApplication:s QTranslator and call `QTranslator::translate(...)`. `QTranslator::translate(...)` will then return a translated string. The good thing here is that `QTranslator::translate(...)` is virtual, so we can overload it!

So by looking at the example above `tr()` is called with `My Button` - so the QTranslator needs to be able to map `My Button` in some way so it fast and simple can lookup the translation for this.

# Let's create an example - Rövarspåret
I'm from Sweden and my childhood has been mark by many of [Astrid Lindgrens](https://en.wikipedia.org/wiki/Astrid_Lindgren) books. In the books about [Bill Bergsson](https://en.wikipedia.org/wiki/Bill_Bergson) the kids are solving crimes. To be able to speak in code they've invented a language called [Rövarspråket](https://en.wikipedia.org/wiki/R%C3%B6varspr%C3%A5ket) (in English - The Rubber Language). It is a fairly simple code. Every consonant is doubled and an 'o' is inserted in between. The vowels are left intact. So Erik is Erorikok. Qt is Qoqtot.

We start with a really simple QML application - a list of fruit emojis.

![Simple application]({{ site.url }}/images/qtranslator-1.png)


```cpp
import QtQuick 2.12
import QtQuick.Window 2.12

Window {
    visible: true
    width: 640
    height: 480
    title: qsTr("Hello World")

    ListModel {
        id: fruitModel
        ListElement { name: QT_TR_NOOP("Grapes"); icon: "1f347" }
        ListElement { name: QT_TR_NOOP("Melon"); icon: "1f348" }
        ListElement { name: QT_TR_NOOP("Watermelon"); icon: "1f349" }
        ListElement { name: QT_TR_NOOP("Tangerine"); icon: "1f34a" }
        ListElement { name: QT_TR_NOOP("Lemon"); icon: "1f34b" }
        ListElement { name: QT_TR_NOOP("Banana"); icon: "1f34c" }
        ListElement { name: QT_TR_NOOP("Pinapple"); icon: "1f34d" }
        ListElement { name: QT_TR_NOOP("Red apple"); icon: "1f34e" }
        ListElement { name: QT_TR_NOOP("Green apple"); icon: "1f34f" }
    }

    ListView {
        anchors.fill: parent
        anchors.margins: 10
        model: fruitModel

        delegate: Item {
            height: 82
            width: ListView.view.widht

            Row {
                spacing: 20

                Image {
                    source: "https://github.com/twitter/twemoji/raw/master/assets/72x72/" + icon + ".png"
                    width: 72
                    height: 72
                }
                Text {
                    text: qsTr(name)

                    height: 72
                    verticalAlignment: Text.AlignVCenter
                    font.bold: true
                    font.pixelSize: 30
                }
            }
        }
    }
}
```
Nothing strange here except for maybe QT_TR_NOOP. What QT_TR_NOOP does is to tell lconvert that this is a string we would like to mark for translation, but when running the code we let the translation happen when we assign the string to the Text-element by using qsTr() - `text: qsTr(name)`.

To make our own QTranslator with Rövarspråket we simply:
```cpp
#include "mytranslator.h"

MyTranslator::MyTranslator(QObject *parent)
    : QTranslator(parent)
{
    vowels << 'a' << 'e' << 'i' << 'o' << 'u' << 'y';
}

QString MyTranslator::translate(const char *context,
                                const char *sourceText,
                                const char *disambiguation,
                                int n) const
{
    Q_UNUSED(context)
    Q_UNUSED(disambiguation)
    Q_UNUSED(n)

    if (sourceText == nullptr)
        return QString();

    QString srcString = QString::fromLatin1(sourceText);
    QString outString;

    for (const auto &c : srcString) {
        if (vowels.contains(c.toLower()) == false && c != ' ') {
            outString += c;
            if (c.isUpper()) {
                outString += 'O';
            } else {
                outString += 'o';
            }
            outString += c;
        } else {
            outString += c;
        }
    }

    return outString;
}
```

This will change the incoming string to a translated version using Rövarspråket.

The last thing we need to do before running the program is to install this translator.
```cpp
QGuiApplication app(argc, argv);

MyTranslator *t = new MyTranslator();
app.installTranslator(t);
```

![Same application but with our translator]({{ site.url }}/images/qtranslator-2.png)

You will find the full source code at [my github](https://github.com/ortogonal/qtranslator-demo).

# Why use your own QTranslator
- By implementing your own you can select the format to store the data in.
- You might have a legacy system or another translating system you "have" to use.
- You need a way to dynamically handle translations and changes in them "live".

Maybe you can name even more in the comments!
# Some other tips regarding translations in Qt/QML
Here are some tips
### Pass information to the translator
You can pass comments to the translators by using special tags while commenting
```cpp
Text {
    //: This is a comment for the translator
    //= 123
    //~ key1 value1
    //~ key2 value2
    text: qsTr("Hello world")
```
As you can see in the above QML-code (it is the same for C++ code) we can pass a comment, an ID and key-value data to the translator by using some special tags.

### QQmlEngine::retranslate()
Since Qt version 5.10 `QQmlEngine::retranslate()` exists. It's a slot that you can call the force the QQmlEngine to retranslate all text dynamically. Really simple. Before this was available you had to play som tricks with the text to get them dynamically translatable. This is no longer the case!

Please leave a comment if you like this!
