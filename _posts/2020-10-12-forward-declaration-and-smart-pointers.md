---
layout: post
title: "Forward declaration and smart pointer in C++"
description: "I preach many thing when talking about programming. But the last years there are two things that is more common in my programming preaching - C++11 smart pointers and forward declaration in header files. But sometimes these two things don’t walk hand in hand. Why?"
tags: [c++, smartpointers]
category: cpp
modified: 2020-10-12
image:
  path: /images/programming.png
  feature: programming.png
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
---
 I preach many thing when talking about programming. But the last years there are two things that is more common in my programming preaching - **C++11 smart pointers** and **forward declaration** in header files. But sometimes these two things don't walk hand in hand. Why?
## C++11 and smart pointers
I've already written a couple of articles about why we shall use them as much as possible, no need to go into that again.

 - [Resource handling with smart pointers](https://ortogonal.github.io/cpp/smart-pointers-architecture/)
 - [Smart pointers in C++11](https://ortogonal.github.io/cpp/smart-pointers/)

## Forward declaration
First of all, what is it?
```cpp
bar.h
#pragma once

class Foo; // <- Forward declaration of Foo
class Bar {
   Bar();
   ...
private:
	Foo *m_foo;
}
```
```cpp
bar.cpp
#include "bar.h"
#include "foo.h"

Bar::Bar() {
    m_foo = new Foo();
}
```
In the above example we do a forward declaration of the class `Foo`. Why?

There are a number of different reasons to do this, but the main being that we reduce the build dependencies between files. Instead of doing forward declaration we could have done a `#include "foo.h"` in the above code. That is just fine, but it adds a build dependency too `foo.h`. If we do a change in `foo.h` the above file will also be treated as "need for rebuild" which might lead to a big chain of rebuilds that is not needed. But in this case we only use `Foo` as a pointer and the only thing we need to know in the header is that we need to reserve space for a pointer, we don't need any more information about `Foo` until we start `m_foo` in the source, and hence we do the `#include "foo.h" in the source file.

Doing this will help speeding up incremental builds when developing code which will reduce frustration from us developers and also save power (no planet B you know)!

## Put the above two things together and you get into trouble
We can more or less take the same example above, but instead of class `Foo` and `Bar` we have `Device` and `FooService`. We also use `std::unique_ptr` (this is the same with `std::shared_ptr` and `std::weak_ptr`).

```cpp
#pragma once

#include <memory>

class Device;

class FooService
{
public:
    FooService();

    void someFunction();
    int getInspiration();

private:
    std::unique_ptr<Device> m_device;
};
```
We forward declare the class `Device` instead of including `device.h` and are happy about that until we compile this code.
```
/usr/include/c++/7/bits/unique_ptr.h:76: error: invalid application of ‘sizeof’ to incomplete type ‘Device’
  static_assert(sizeof(_Tp)>0,
                      ^
```

> WTF! Thanks for that error message.

You start to think!
>This should work, it's just a pointer.

>This has worked before.

So let's break this down. You might have written code just like this and got away with it. You might give up and just do the `#include "device.h"` because it needs the type information.

### The real problem here
The real problem here is that we let `std::unique_ptr` handle the deletion of m_device using its default deleter function (which is great, the hole idea with `std::unique_ptr`). But the `std::default_delete` doesn't know how big the class `Device` is at this point so how can it know what to delete?

Looking at the C++ reference documentation for `std:unique_ptr` it says:

> `std::unique_ptr` may be constructed for an incomplete type `T`, such
> as to facilitate the use as a handle in the Pimpl idiom. **If the
> default deleter is used,  `T`  must be complete at the point in code
> where the deleter is invoked, which happens in the destructor, move
> assignment operator, and reset member function of
> `std::unique_ptr`**. (Conversely, `std::shared_ptr` can't be
> constructed from a raw pointer to incomplete type, but can be
> destroyed where `T` is incomplete).

### The simple solution
So what does this mean? Well simple - create a destructor!
```cpp
#pragma once

#include <memory>

class Device;

class FooService
{
public:
    FooService();
    virtual ~FooService();

    void someFunction();
    int getInspiration();

private:
    std::unique_ptr<Device> m_device;
};
```
```cpp
#include "device.h"
...
FooService::~FooService() {}
```
By just creating an empty destructor in `fooservice.cpp` where we included `device.h` we now have "created the place" were `std::default_delete` runs and then it knows the size of `Device`. This is we you might have written code that works because you had a destructor.

This was a short and simple post. Hope you like the tips!
**Thanks for reading!**
