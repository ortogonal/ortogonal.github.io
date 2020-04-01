---
layout: post
title: "Better resource handling, less gray hair in C++"
description: "Using smart-pointer for better C++ architecture"
tags: [c++, cpp, modern c++]
category: cpp
modified: 2020-04-01
image:
  path: /images/lemmons.jpg
  feature: lemmons.jpg
---

The last couple of years I've been getting more and more interested in making C++ simpler to maintain and use. Resource handling in C++ is one issue were I (and others) have been struggling. Struggle that cased bugs, frustration and maybe a few gray hairs. In this article I'll try to explain my *"rules"* to sort out this resource struggle by using smart-pointers.

In the end this is also about my quest for getting rid of naked `new` and `delete` from the source, but also to make sure we have a clear view on resource ownership.

In this post I will cover
 - Ownership of resources
 - When to not pass a smart-pointer
 - Why `const`

# No raw-pointer and references as member variables
It is a pretty common case to pass raw-pointers or references into a class constructor to create some sort of aggregation or composition which creates a dependency. Personally I think that this is not the best way to develop when C++11 have the smart-pointers.

The problems with raw-pointers and references as member variables:
 1. There is no way of knowing if this is a composition or aggregation, in other words - no way of known who owns the resource.
 2. The class that holds the raw-pointer/reference can't guarantee that the raw-pointer/reference will outlive the class itself, it is up to the developers to keep track on.
 3. If there is a need for a pointer, that is only an internal pointer in that `class`, just use a `std::unique_ptr<>` to get rid of the raw-pointer and you can get rid of the deallocation in the destructor for the `class` and hopefully get rid of the destructor it self.

So start with getting rid of the raw-pointers and references as member variables and replace them with smart-pointers.

## Solving the owner problem.
In a `class` you can have three types of ownership - a unique-, a shared- and a weak ownership (weak ownership is also called weak-reference). With smart-pointers we can explicitly select the ownership in the code, without them it's not possible.

The code below illustrate the danger with raw pointers as members.
```cpp
class Foo {
public:
    Foo (Bar *bar) { m_bar = bar; }
    void doWork() {
        if (m_bar != nullptr)
            m_bar->doWork();
    }
private:
    Bar *m_bar = nullptr;
};
```
There is no way in the above code to know anything about the ownership of `m_bar`. Is it shared, is it unique? A potentially dangerous situation can be this.

```cpp
Bar b = new Bar();
Foo f(b);
delete b;
// b is deleted, but calling f.doWork() will
// call Bar::doWork() by using a deleted pointer.
f.doWork(); // <- Segmentation fault!
```
But if we switch to smart-pointers we can solve this problems. Now let's look at how we create a specific ownership using smart-pointers in C++.

### Unique ownership
```cpp
class Foo {
public:
    Foo (std::unique_ptr<Bar> bar) {
        m_bar = std::move(bar);
    }
private:
    std::unique_ptr<Bar> m_bar;
};
// Create f which has a unique ownership now
Foo  f(std::move(std::make_unique<Bar>()));
```
The above code **has a unique** ownership now and it will generate compile time errors if you do not move your `std::unique_ptr<Bar>` into the class `Foo`. Also this is readable by the developer. There is no question about the ownership in this case, The class `Foo` has the unique ownership and responsibility of `m_bar` at this point.

### Shared ownership
```cpp
class Foo {
public:
    Foo (std::shared_ptr<Bar> bar) {
        m_bar = bar;
    }

private:
    std::shared_ptr<Bar> m_bar;
};

Bar bar = std::make_shared<Bar>();
Foo f(bar);
```

In the above example the class `Foo` is changed to have a shared ownership. We are using a `std::shared_ptr` now which means that we can guarantee that `m_bar` will be available as long as our instance of `Foo` is available. We also know that all resources of `Bar` will be released when no one needs it any more. That means that `bar` can be passed to other classes or used in any other way. The shared ownership is also a really nice way of replacing lazy singleton solutions from your code.

### Weak ownership
First of all, weak ownership what is that for kind of ownership? Well a ownership were the resource might be available but you need to make sure it is before using it. But when you use it, you know that it's available. This is exactly what `std::weak_ptr` does.

```cpp
class Foo {
public:
    Foo (std::weak_ptr<Bar> bar) {
        m_bar = bar;
    }

    void doWork() {
        if (!m_bar.expired()) {
            std::shared_ptr<Bar> bar = m_bar.lock();
        }
    }
private:
    std::weak_ptr<Bar> m_bar;
};

Bar bar = std::make_shared<Bar>();
Foo f(bar);
```
Now we have modified `Foo` so that it has a weak ownership (or a weak reference). This means that `m_bar` can be expired and not valid any more which is something we must check before we use it. This is done by:
```cpp
if (!m_bar.expired()) {
    std::shared_ptr<Bar> bar = m_bar.lock();
    bar->doWork();
}
// OR
// This is the same as above but shorter
if (auto bar = m_bar.lock()) {
    bar->doWork();
}
```
When using `std::weak_ptr` you assign a `std::shared_ptr` to it (as value). Later when you use your `std::weak_ptr` you must make sure it's still valid. This is done by either calling `std::weak_ptr::expired()` or by calling `std::weak_ptr::lock()` and look if the returning pointer is not `nullptr`.

After calling `std::weak_ptr::lock()` we have a `std::shared_ptr` that we can act on and that will stay valid at least until we release it, maybe longer depending on if other holds any reference to it. Using `std::weak_ptr` is a really useful when working with caches for example. But there are many other use-cases for it!

### Ownership - summary
The above examples of different ownership shows that by removing raw-pointers and references from your member variables and instead replacing them with smart-pointers your architecture becomes safer (in that sense you get compile errors) and much clearer for the other developers looking at your code.

By using smart-pointers we now have a clear picture of which ownership is used. Also the ownership is automatically maintained by the smart-pointers and not by us developers!

```cpp
Foo (std::unique_ptr<Bar> bar) // Unique ownership
Foo (std::shared_ptr<Bar> bar) // Shared ownership
Foo (std::weak_ptr<Bar> bar) // Weak reference
```
The above example show that by just looking at the signature of the constructor/function we can know which ownership type is used.

# When to not pass a smart-pointer
In the above section I write that all raw-pointer shall be removed from member variables, but that is not the same thing as raw-pointers (and references) are bad and ugly. In contrary they fulfill the next step in making the architecture clearer.

But first of all we need to setup a number of ground guide-lines/rules to work with.
 1. **Never ever** convert a raw-pointer (or reference) that is passed as a function argument into a smart-pointer by using `std::unique_ptr<>()` or `std::shared_ptr<>()`.
 2. If a function has a reference as parameter the caller is responsible for making sure that reference is valid.
 3. If a function has a raw-pointer as parameter the caller can pass `nullptr` into said function so the function must make sure that the pointer isn't `nullptr`.
 4. Try to use rule 2 over rule 3.

## Rule 1 - Never create a smart-pointer from a raw-pointer (or reference)
I cannot emphasize this enough. I my opinion you shall **never** convert a raw-pointer into a smart-pointer if the raw-pointer is passed as a function argument. This will lead to double deletes and frustration.
```cpp
class Foo {
public:
    void setBar(Bar *bar) {
        // Don't do this!!!
        // Please don't!!!
        m_bar = std::shared_ptr<Bar>(bar);
    }

private:
    std::shared_ptr<Bar> m_bar;
};

Bar bar;
auto foo = std::make_unique<Foo>();
foo->setBar(&bar);
```
The above example illustrates the risk of converting raw-pointers into smart pointers without letting the developer really know. The code above will cause a double release and a crash.

## Rule 2 - Reference as parameter
```cpp
class Foo {
public:
    int calculateLength(const Bar &bar) const
    {
        return bar.length() + length();
    }
};

Bar b;
Foo f;
f.calculateLength(b);
```
If we design our code with this rules in mind the caller of `Foo:calculateLength(const Bar &bar)` knows that there is no ownership change (thanks to rule #1) or any other strange thing related to the reference that is passed. And also the developer of the class `Foo` knows that it doesn't need to check that the argument `bar` is valid.

## Rule 3 - Raw-Pointers as parameter
```cpp
class Foo {
public:
    int calculateLength(const Bar *bar) const
    {
        if (bar != nullptr)
            return bar->length() + length();
        else
            return 0;
    }
};

Bar b;
Foo f;
f.calculateLength(&b);
f.calculateLength(nullptr);
```
In this case we have a complete different situation. Instead of a reference we pass a raw-pointer. Now the caller can pass a `nullptr` and it's the functions responsibility to check whether the argument `bar` is valid or not.

## Rule 4 - Rule 2 over rule 3
Try to pass by reference instead of raw-pointer, but still make sure to follow the rules setup here.

## When to not pass a smart-pointer - summary
The above four rules will, together with the the ownership handling when using smart-pointers, create an API that is clear for the user just by examine the function signatures. This helps the developer of knowing that data flow in the system.

Unfortunately there is now way of getting compile time errors on the above four rules. Maybe this is something one could solve by creating a couple of self-designed test in Clang-Tidy! The future will tell. But these rules can be verified during development and during code review. They are pretty simple to follow.

#  Make sure to use `const` everywhere it's needed
You cannot have to much `const`.

I listen to a talk from CppCon (can't remember which) were the speaker proposed that C++ should have everything `const` as default and when you don't need const you had to mark it mutable. This is more or less impossible to add right now in the C++ language, but this is something we developer can have as a mindset.

`const` helps us not change things we're not suppose to change. `const` gives us compiler error if we try. `const` is something that a developer can use to *"tell"* his/hers intentions to another developer.

## `const` when passing raw-pointers and references
This is probably the most common case where you see `const` used, when passes as function variables.
```cpp
int Foo::totalCost(Bar &bar);
int Foo::totalCost(const Bar &bar);

int Foo::totalCost(Bar &bar) const;
int Foo::totalCost(const Bar &bar) const;
```
The above code shows four different functions which all do the same. But it's only `int Foo::totalConst(const Bar &bar)` and `int Foo::totalConst(const Bar &bar) const` were we for sure knows that Bar is not going to be affected by calling this function.

The third variant, `int Foo::totalCost(Bar &bar) const`, is still `const` as a function in regards to the class `Foo`, but `bar` is still **not** `const`, so we can still do none-`const` operations on `bar`. This is something we don't want (in most cases). Instead use the last alternative if everything needs to be `const`.

When designing API:s using `const` is something that will help other developers know the intentions of a function of class. `const` is something that I thing is really essential in your API:s.

## `const` ownership
A `const` ownership might sound strange. Constantly owning something? - No that is not what it's about. But when having `std::shared_ptr` or `std::weak_ptr` as member variables they might need to be `const`, which generate a `const` ownership.
```cpp
class Foo
{
public:
    Foo(const std::shared_ptr<Bar> bar) : m_bar(bar) {};
private:
    const std::shared_ptr<Bar> m_bar;
}
```
The code above show that the class `Foo` has a shared ownership of `bar`, but in this case the developer also are telling us that it's `const`. `Foo` will (and cannot) do any none-`const` operations on `bar`. This can be useful to when a class only are allowed to observe `m_bar` (in this case). It's maybe not the most important thing but can still be useful.

# Summary
These rules and API design ideas is pretty simple to follow and the benefit of using them is that you get code that is simpler to read, simple to understand how resources are handled.

This doesn't solve all problems when building big and complex C++ software - but it makes it clearer and easier to understand. Lately I've tried to follow them as much as possible while writing code and I really like them. It give me a clear picture over how ownership, something that is often forgotten.

Please leave a comment if you have other ideas, suggestions or simple doesn't agree with me.
