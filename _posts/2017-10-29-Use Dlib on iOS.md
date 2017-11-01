---
layout: post
title: Use dlib on iOS project
description: "Keep records of how I did it."
modified: 2017-10-29
tags: [dlib, iOS, xcode]
image:
  path: https://github.com/XiomarG/blog/blob/gh-pages/images/abstract-7.jpg
  feature: abstract-7.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
---

I started an iOS project last week and decided to use dlib for some features it provides.
This blog will keep track of the process.

## Compile with dlib
There is an introduction blog about this topic.
[Getting Started With dlib on iOS](http://prabhu.xyz/2017/05/29/getting-started-with-dlib-on-ios.html)

It was very useful but while following it, I still got some problems.

### 0
Some error about libdlib.a not compiled for arm64.   
Solution: 
don't use **General iOS Device** which uses mac's architecture. Use simulator or real device.

### 1
```
Undefined symbols for architecture arm64:
  "_USER_ERROR__missing_dlib_all_source_cpp_file__OR__inconsistent_use_of_DEBUG_or_ENABLE_ASSERTS_preprocessor_directives_", referenced from:
      _dlib_check_consistent_assert_usage in DlibWrapper.o
ld: symbol(s) not found for architecture arm64
clang: error: linker command failed with exit code 1 (use -v to see invocation)
```
Solution:
https://stackoverflow.com/questions/37208964/error-including-dlib-library-in-c-project
add dlib/all/source.cpp to your project sources list(CXXFILES)

### 2
```
jpeglib.h file not found
```
Initially I tried to look for where the <jpeglib.h> was included. I found that in the tutorial it set preprocessor macro DLIB_JPEG_SUPPORT. And in the code, it said 
```
#ifdef DLIB_JPEG_STATIC
#   include "../external/libjpeg/jpeglib.h"
#else
#   include <jpeglib.h>
#endif
```
I thought it was because that I didn't compile dlib with this `DLIB_JPEG_STATIC` set. But I was wrong.
Anyway, after about 10 hours try/error, I released that my dlib archive file was in x86_64 architecture. I then went back to the tutorial and read line by line.

After 
```
cmake -G Xcode ..
cmake --build . --config Release
```
There is a libdlib.a in Release folder. This is not the one I suppose to use. The architecture was by default macos.
I should open the Xcode project in build/dlib_build, change the project setting (not the target setting!), use architecture = arm64, create new schema for release, then run it. Now there is a new folder called `/build/dlib_build/Release-iphoneos`. Inside it there is a new libdlib.a. This is the one I should use.
*hint: at this time the error was actually `symbol not found for arm64.... libdlib.a is built for architecture other than arm64...` something like that.*