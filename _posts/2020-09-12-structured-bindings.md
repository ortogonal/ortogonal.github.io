---
layout: post
title: "Structured bindings in C++17"
description: "Create cleaner code with structured bindings"
tags: [c++, cpp, modern c++17]
category: cpp
modified: 2020-03-10
image:
  path: /images/abstract-3.jpg
  feature: abstract-3.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
---


Many people argue that C++ is an old outdated language when compared to more modern ones like Python, Go, Rust and many more. Sure, in some cases this might be true. But I strongly feel that many times when C++ is compared against other languages it's compared with what C++ was around the 90's and not what it is now.

So this post is both for clearing my head* and showing what structured bindings in C++17 will let us do.

## Why structured binding?
Well, finally we can return more than one parameter from a function! Blog post is done! Thanks for reading, see ya next time...

But there is more to it, so keep on reading.
Even though multiple return-values was the reason I started to look into structured bindings it doesn't mean it is done there. There are a number of places were structured bindings will help you write cleaner C++ code.

## Examples on where to use it
I'll list a number of different places were structured bindings can be used, some might be better than others. The first two are the ones I really like, the last two might not be something I will use every day, but might be good to know about.

### Multiple return values
I walk in the park to return multiple values from a function! Just return a `std::tuple` then use `auto[]` to handle the values.

In the example below we create the return values using `std::make_tuple()`. A `std::tuple` can contain how many elements you like so not limited to three as in this example.
```cpp
#include <iostream>
#include <tuple>

using namespace std;

std::tuple<int, std::string, float> foo()
{
    return std::make_tuple(1, "Hello world!", 3.14f);
}

int main()
{
    auto [line, str, pi] = foo();
    cout << str << endl;
    return 0;
}
```

In the the above example we use `auto [line, str, pi]` to decomposing the variables. This is the simplest solution and requires the least amount of writing. But it requires you to define the variables, even if you are just interested in two of them.

One solution to that is using `std::ignore`. The problem with that is that you instead need to use `std::tie`.

```cpp
int line;
std::string str;
std::tie(line, str, std::ignore) = foo();
```
The above snippet shows how to replace `auto[]` with `std::tie`. The only difference is that you need to define the variables before, but instead can use `std::ignore`. So more writing :(

Happy returning!

### Iterating over maps
Next up - `std::map`!

Using range-loops when dealing with `std::vector` or `std::list` have made the C++-code much cleaner. But `std::maps` has been a bit tricky to handle with range-loops. But not with C++17!

```cpp
std::map<int, std::string> myMap;

myMap.insert({1, "Apple"});
myMap.insert({2, "Orange"});
myMap.insert({2, "Lemon"});

for (auto &[id, fruit] : myMap) {
	std::cout << id << " " << fruit << std::endl;
}
```
In the above code we create a simple `std::map<int, std::string>` and inserts some fruits. The last thing we do is to iterate over the map by using `auto &[id, fruit]`. This creates a super-simple range based loop with a really clean look!
I really like using `std::map` when coding. But the missing part of `std::map` has always been the range based loops. With C++17 it's solved!

### Array
Well, is this really something good? I don't know.
But you can use structured bindings together with arrays.
```cpp
int arr[3] = { 1, 2, 3};
auto [x, y, z] = arr;

std::cout << x << y << z << std::endl;
```
The above code will bind the first value in the array to `x`, the second to `y` and so on. This looks pretty okay at first glance. But you end up with the same problem as with return values from functions - the amount of variables must match with the array. So it might be a bit tricky to use.

### Structs and variables
You can use structured bindings together with `struct`. Just dive into the example.
```cpp
struct Point {
    float x;
    float y;
    float z;
};

Point p { 1.f, 2.5f, -0.4f };

auto [fX, fY, fZ] = p;
std::cout << fX << " " << fY << " " << fZ << std::endl;
```
As you can see we can create the variables fX, fY and fZ that we bind to the `Point` x, y and z. But really why? Is it that hard to write `p.x`? Please leave a comment and enlighten me if so!

## Conclusion
I like to use structured bindings for `std::maps` and return values for functions. It really creates cleaner code that is simple to read. What do you think? Leave a comment and share your thoughts.

See ya!



**The passion for coding doesn't always match your work, but never let someone take away your passion.*
