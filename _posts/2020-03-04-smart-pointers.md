---
layout: post
title: "When the company porn filter stopped me from programming"
description: "Get rid of naked new and delete with C++ smart-pointers"
tags: [c++, cpp, modern c++]
category: cpp
modified: 2020-03-10
image:
  path: /images/cpp-smart-pointers-header-2.png
  feature: cpp-smart-pointers-header-2.png
---


This is new - I tried to push some code the our git server and the company porn filter did not allow it! After spending some time a found the problem in my code
```cpp
ObjectType *type = new ObjectType();
```
The porn filters now a days are so sophisticated that the can find naked pointers! Wow!

## No that did not happen - it's just a start of the article
But one thing that happen was that I realize that many still uses naked pointer in C++. Why? Well maybe simply because they don't know of `std::unique_ptr<>`, `std::shared_ptr<>` and `std::weak_ptr<>` (not covered in this article)?

## The problem with naked pointers - you and me
There is nothing wrong with the pointers them self, the main problem is you and me. We tend to forget to deallocate what we allocated. We pass pointers around without having a clear picture who is the owner of it and if it might outlive another objects that handles it. This results is crashes and/or memory leaks.

So to solve this we can use these "Smart Pointer" that will help us with resource allocation and deallocation. They also helps us handle ownership of our pointers.

## The syntax of smart pointers
This is the simple part. The syntax for using a smart pointer is the same as for a raw pointer.
```cpp
std::string *rawPtrString = new std::string();
std::shared_ptr<std::string> smartPtrString = std::make_shared<std::string>();

if (rawPtrString != nullptr)
    std::cout << rawPtrString->size() << std::endl;
if (smartPtrString != nullptr)
    std::cout << smartPtrString->size() << std::endl;
```
The `->`-operator is used to access the pointer member functions/variables. You can also use the smart pointer to compare against `nullptr`.

To access the raw-pointer one can use `.get()`, to release the pointer one can use `.release()`. I will not go more into detail about the specific member functions available. More info can be found [here](https://en.cppreference.com/w/cpp/memory/unique_ptr)

Maybe in some other article I will cover them more deeply.

## `std::unique_ptr<>` - The most simple one
The `std::unique_ptr<>` is the simplest smart pointer. It hold a pointer and it's unique. It will deallocate the pointer when the variable goes out of scope. Here is an example

### Allocation/Instantiation
```cpp
#include <memory>

class Foo
{
public:
    Foo()
    {
        m_bar = std::make_unique<Bar>();
        std::unique_ptr<Bar> anotherBar = std::make_unique<Bar>();
    }

private:
    std::unique_ptr<Bar> m_bar;
};
```
This example has two `std::unique_ptr<>`:s, `m_bar` as a member variable in `Foo` and `anotherBar` as a variable in the constructor of `Foo`. Both `m_bar` and `anotherBar` is created in the constructor.
```cpp
m_bar = std::make_unique<Bar>();
std::unique_ptr<Bar> anotherBar = std::make_unique<Bar>();
```
Both are using `std::make_unique<>` to allocate the new Bar-objects on the heap memory. The difference between `m_bar` and `anotherBar` is simply that `m_bar` will deallocate `Bar` when the `Foo` object is deallocated and `anotherBar` is deallocated when exiting the constructor (it's scope).
So what `std::unique_ptr` does is simple deallocating it's resource when exiting it's scope!

`std::make_unique<>` is a C++14 feature. If using C++11 you would have to use `m_bar = std::unique_ptr<Bar>(new Bar())`.

### Why you can't copy a `std::unique_ptr`
By design a `std::unique_ptr` isn't copyable. Nothing strange with that when you think about it. If you would copy something that is unique, it's not unique any more.

But instead of copy we can **transfer ownership** by using the move-semantics.

```cpp
class Foo
{
public:
    Foo(std::unique_ptr<Bar> bar) {
        m_bar = std::move(bar);
    }

private:
    std::unique_ptr<Bar> m_bar;
};

int main(int argc, char *argv[])
{
    auto bar = std::make_unique<Bar>();
    Foo f(std::move(bar));
}
```
As you can see in the example above `std::move(bar)` is use to **move** the ownership from one to another.

## `std::shared_ptr<>`  - pointer with build in ref. counter
The difference between a `std::unique_ptr` and `std::shared_ptr` is that the `std::shared_ptr` has a shared ownership instead of the unique ownership that `std::unique_ptr` has.

So what does shared ownership mean? Simply put it the last owner is the one making sure the the resource is deallocated. In [*Arthor O'Dweyers* talk about smart-pointers at CppCon](https://www.youtube.com/watch?v=xGDLkt-jBJ4) he had an analogy about the last person exiting a room is responsible for turning the lights off.

*The rules for this shared responsibility (turning the light of) is simple. When you enter the room you put a token into a jair, when you exit you take one token out. If the jair is empty when you take you token out - you turn the lights of.*

`std::shared_ptr` works in the same way. For every shared-owner that exists the reference counter is increased and when the pointer loose a shared-owner it decrease the reference counter. If the counter is zero it will deallocate the pointer.

This creates a way of having a shared ownership of pointers. You can create a `std::shared_ptr` and pass it to another object that also holds a reference to it and no longer worry about when the resource is deallocated.

```cpp
class Foo
{
public:
    Foo(std::shared_ptr<Bar> ptr) {
        m_bar = ptr;
    }

    void work() {
        m_bar->work();
    }

private:
    std::shared_ptr<Bar> m_bar;
};

int main(int argc, char *argv[])
{
    auto bar = std::make_shared<Bar>();
    auto foo = std::make_unique<Foo>(bar);

    foo->work();
    // Reset bar
    bar.reset();

    // Still okay, because m_bar still has a
    // reference to bar created in the beginning
    foo->work();
}
```

## Why use `std::make_unique` and `std::make_shared`?
```cpp
A *a = std::unique_ptr<A>(new A());
auto a = std::make_unique<A>();
```
Why is the later better. Well two things. First of all `std::make_unique<>()` is a bit more optimized, but let's not get into that. Second and the more important reason to use `std::make_unique<>()` and `std::make_shared<>()` - You can end up with code that doesn't contain any `new` or `delete`.

When I first learned C++ the thing you should do was to make sure to add your `delete` that "matched" the `new` you just added. But now we don't need `delete` anymore so let's get rid of `new` also and get rid of that said rule!

## Custom deleters
Both `std::unique_ptr` and `std::shared_ptr` handles defining a custom deleter. This can be used for customizing which code needs to be executed for releasing the resources. This is especially be useful when working with many C-libraries.

For example when working with libLXC you use `lxc_container_new(...)` and `lxc_container_put(...)` to allocate and deallocate a container. Using smart-pointers this can be done by:
```cpp
struct LXCDeleter
{
    void operator()(lxc_container *ptr) const {
        lxc_container_put(ptr);
    }
};

auto filePtr = std::unique_ptr<lxc_container, LXCDeleter>{ lxc_container_new("foo", nullptr) };
```
The above example will create a `std::unique_ptr<>` by using `lxc_container_new("foo", nullptr)` to allocated the container. When the unique_ptr goes out of scope it will use the custom deleter `LXCDeleter`  which will call `lxc_container_ptr(ptr)` to deallocate!

Another example is when working with files.
```cpp
struct FileDeleter
{
    void operator()(FILE *ptr) const {
        fclose(ptr);
    }
};

auto filePtr = std::unique_ptr<FILE, FileDeleter>{ fopen("foo.txt", "rw") };
```
The above code will make sure to close the file when the `std::unique_ptr` goes out of scope.

## Whats next?
Well you tell me. I got some ideas to continue write about `std::week_ptr<>` or maybe go into more detail one how to reference-counting works, or how to design good API:s with smart-pointers. There is also the fun `std::enable_shared_from_this` that can be something to write about.

But most important - start writing code with no naked `new` or `delete`.

Also spam your company porn filter with this.
<iframe src="https://open.spotify.com/embed/track/4YidEz8xGN9NZeUXFc94yw" width="300" height="380" frameborder="0" allowtransparency="true" allow="encrypted-media"></iframe>
