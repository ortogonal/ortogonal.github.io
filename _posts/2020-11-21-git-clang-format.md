---
layout: post
title: "Format your code - all the time"
description: "Use git-clang-foramt to format (and enforce format) while hacking away"
tags: [c++, cpp, modern c++]
category: cpp
modified: 2020-11-21
image:
  path: /images/git-clang-format.png
  feature: git-clang-format.png
---

Formatting source code is something that, at least I think, is important to raise the code quality. I meet many developers that don't have the same strange passion for it though. But the reason formatting your code is important is that you get a uniformed code when reading it, meaning you will decrease the "WTF per minute"-score*.

This article is not about whether a `{` should be on the same line of the `if-case` or on the line below. This article is about the tools and method to use to always use the same formatting.

## clang-format
Out of the [LLVM/Clang](https://clang.llvm.org/docs/ClangFormat.html) open source project the tool `clang-format` was created. The idea of `clang-format` is simple. You setup a [configuration file](https://code.qt.io/cgit/qt-creator/qt-creator.git/tree/.clang-format) that defines the format you like to use on your code and the run `clang-format`. `clang-format` will then reformat your code into something that follow your rules in the configuration file. In `clang-format` there are options for setting more or less everything. Normally you store your settings in a file called `.clang-format`, then `clang-format` will use it. It handles code written in C/C++/Java/JavaScript/Objective-C/Protobuf/C#.

### Example
```cpp
void test(QString&data, bool extraString) {
    int i=0;
    for (i=0;i<3;i++) {
        data+="reallylongstringtoproducealonglineasanexample" + QString::number(i * 1000) + "/filetoload.html";
        if (extraString)
        {
            data += "some-extra";
        }
    }
}
```

To format your code run:

{% terminal %}
$ clang-format -i mysource.cpp
$
{% endterminal %}

After running `clang-format` we get this
```cpp
void test(QString &data, bool extraString)
{
    int i = 0;
    for (i = 0; i < 3; i++) {
        data += "reallylongstringtoproducealonglineasanexample" + QString::number(i * 1000)
                + "/filetoload.html";
        if (extraString) {
            data += "some-extra";
        }
    }
}
```
The above code is completely useless, but it illustrate what clang-format can do to your code. If you like this formatting or not is not of importance here, because you can setup any rules that you and your team like! Since I write a lot of code together with the Qt framework I've continued using the `.clang-format` [file used in QtCreator](https://code.qt.io/cgit/qt-creator/qt-creator.git/tree/.clang-format).

### Problems with `clang-format`
`clang-format` is a really nice tool to use but it has one problem - **legacy code** and your **commit history**. The current project I'm working on has a pretty huge code base which is formatted without any real *"rules"* from the beginning. When I stepped into the project I started to push for using clang-format.

The problem we have is simple. When reformatting a source file we end up with **tons of changes** that destroys the commit history and makes merging of branches harder, it also makes reviewing and code archaeology** much harder because you have to look at the changes in the code and first figure out if this is a change of the code or if it's just a formatting change that actually doesn't do any thing.

Another problem is maintaining the code format when adding new into already formatted code. But there is just the tool for this - `git-clang-format`!

## git-clang-format
`git-clang-format` is a simple Python script distributed together with clang-format. The problem is that not so many are talking about git-clang-format.

What `git-clang-format` solves for us is that it runs `clang-format` on the changes you made. This means that we can solve the problems I describe above. Every time we change code it can be formatted with clang-format. Our legacy code base we change in will eventually get better and better formatted code without loosing the readability when reviewing code!

The work flow using `git-clang-format` is pretty nice
* Develop your code
* Run `git clang-format` (no I did not miss a `-` `git-clang-format` can be called from `git` using `git clang-format`. 

This will make sure the changes you have done are correctly formatted!

### Using git-clang-format with a pre-commit hook?
`git-clang-format` is a really nice tool that you can use together with a `git` pre-commit hook. This means that you can setup a hook that makes sure your code is formatted before doing a commit.

#### Example pre-commit hook
Below is a simple example of a pre-commit hook. This script shall be named `pre-commit` and placed in your `.git/hooks`-folder.
```bash
#!/bin/sh  
  
if git rev-parse --verify HEAD >/dev/null 2>&1  
then  
against=HEAD  
else  
# Initial commit: diff against an empty tree object  
against=4b825dc642cb6eb9a060e54bf8d69288fbee4904  
fi  
  
# Test clang-format  
clangformatout=$(git clang-format --diff --staged -q)  
  
# Redirect output to stderr.  
exec 1>&2  
  
if [ "$clangformatout" != "" ]  
then
    echo "Format error!"
    echo "Use git clang-format"
    exit 1
fi
```
**Note:** In the above script I use `--staged` when calling `git clang-format`. The `--staged` option makes sure it only looks at staged code and not all changes. The problem is that, when writing this, it's not part of the official git-clang-format script. It's still in a pull-request I did for clang-format waiting for review. You can find the script [here](https://reviews.llvm.org/D90996).

The above is really nice. Instead of just printing "Format error" this is what I have done in my script to make hacking funnier.

{% terminal %}

$ git commit
______ ______________  ___  ___ _____ 
|  ___|  _  | ___ \  \/  | / _ \_   _|
| |_  | | | | |_/ / .  . |/ /_\ \| |
|  _| | | | |    /| |\/| ||  _  || |
| |   \ \_/ / |\ \| |  | || | | || |
\_|    \___/\_| \_\_|  |_/\_| |_/\_/
 _________________ ___________        
|  ___| ___ \ ___ \  _  | ___ \
| |__ | |_/ / |_/ / | | | |_/ /
|  __||    /|    /| | | |    /
| |___| |\ \| |\ \\ \_/ / |\ \ 
\____/\_| \_\_| \_|\___/\_| \_|
                                   

The new code that is added contains differences 
against clang-format rules. Please fix it before
doing a commit!

{% endterminal %}

A pre-commit in `git` is a nice thing to have to make formatting easier. But remember that it can be by-passed using `--no-verify` when calling `git commit`.

### Rewriting history
A big pull-request the other day inspired to dig into how to solve formatting on bigger blocks of changes. The method described below can be used on a single branch with a number of commits that needs formatting, or **all** your commit.

**WARNING:**  Using `git filter-branch` is not something I recommend doing in front of friends, I also recommend thinking this through before and not just trust a random blogger you found using Google! `git filter-branch` can mess things up.

Okay, hope you read the warning! Now time to play!

Lets say we have a branch with 10-20 commits that needs formatting. We like to do this as pros by keeping the commit history, commit dates and authors. To solve this we need to:
* Check out commit by commit, then
* Run git-clang-format on the changes
* Take the changes and amend them to the original commit

This is something `git filter-branch` can do for you.

#### Example
We have a branch called `feature/this-is-it`. It has 10 commit on top of the `master`-branch. To do all the steps above simple run.

{% terminal %}
$ export FIRST_COMMIT=$(git rev-list --ancestry-path origin/master..HEAD | tail -n 1)
$ git filter-branch --tree-filter 'git-clang-format --extensions h,cpp $FIRST_COMMIT^' -- $FIRST_COMMIT..HEAD
{% endterminal %}

The above example will fetch your `FRIST_COMMIT` from where the `origin/master` is (can be any other branch or commit). The second line will run `git filter-branch` and for each commit run `git-clang-format` and run clang-format over all changes from your `FRIST_COMMIT` and the store that change. The result is a branch where you keep your history but have nicely formatted code!

## Summary
I hope this can help you keep your code well formatted and limit your WTF-per-minute score. Leave a comment if you think I missed anything or if there are other tools you use!

##### * WTF per minute score
*The best way of measuring code quality is to count the number of "WTF" (What the fucks) you think in your head when reading the source. Then count the number of time you say WTF out loud and multiply that number with 4. Next step is to sum the two values and divide the sum with the time you spent reading. The you have your WTF-per minute score. It should be low!*
##### *Code archaeology
*You know when you dig deep down into old commits to try to under stand why a change was made 13 month ago.*

