---
layout: post
title: How to build WebRTC for Mac-Catalyst or Ninja intro for Apple-developers
tags: ninja catalyst marzipan xcode cdnbye
---

**WebRTC** is a de-facto standard for conferencing and peer-to-peer data exchange. It's mature, highly optimized, and build-in into major browsers. For most platforms, it's easy to embed into your application, since WebRTC's team ships binary builds in various formats.

However, supporting various build configurations on this scale is hard and it's not feasible to support all platforms. In this blog, I'll describe what it takes to build it for Mac-Catalyst platform. It's also pretty interesting to learn how advanced build-systems work, just in case you might have to deal with it at some point. I certainly did enjoy playing with it.


Mac-Catalyst is a bit of niche platform, not even major vendors have implemented its support (looking at you Firebase). Essentially, it's an x86_64 arch just like macOS, but not exactly compatible. 
You can, in fact, codesign a mac framework, bundle it and dlopen at runtime - and that will work. But that is non-optimal and requires various hacks to use it since you can't directly reference symbols from it. The proper way is to re-build it for Catalyst.

## Ninja
WebRTC is developed using same build system as Chrome, called [Ninja](https://ninja-build.org/). Ninja was build to address slow incremental build times of other tools and it really shines. I hope to try it one day on a real task, build times is one of the most frustrating things in my day-to-day experience.

So how does it build exactly?
After you get sources and dependencies as outlined in [this](http://webrtc.github.io/webrtc-org/native-code/ios/) guide, you are suggested to run:

```
gn gen out/ios_64 --args='target_os="ios" target_cpu="x64"'
```

GN is a complimentary utility that generates build-description for Ninja.
Now, looking at the `out/ios_64`, we can deduce that `gn` has generated compile & link commands for all modules and files. Ninja's philosophy is to split all sources files into a lot of small pieces, so to make less work when doing incremental builds.

Poking at one of the numerous `.ninja` files, we see familiar cflags, ldflags, and of course`-miphoneos-version-min` & `-isysroot`. So that's our target, if we change those to Catalyst-specific - we are done.
If we examine Xcode's output for Catalyst build,

It has special flags for Catalyst:
```
--target=x86_64-apple-ios13.0-macabi

-isysroot=/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.15.sdk


-iframework /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.15.sdk/System/iOSSupport/System/Library/Frameworks

```

In other words, we need to change GN's source files to include those defines instead.

The most obvious place to start is ./BUILD.gn but it contains just a few flags. Nonetheless, all flags added here should be reflected on each build rule, so I've added
```
+  cflags += [ 
+    "-iframework", "/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.15.sdk/System/iOSSupport/System/Library/Frameworks",
+    "-isystem", "/Applications/Xcode-beta.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.15.sdk/System/iOSSupport/usr/include"
+  ]
+  ldflags += [
+      "-iframework", "/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.15.sdk/System/iOSSupport/System/Library/Frameworks"
+  ]
+
```
here.

But how do we change target ios -> catalyst? e.g. minphoneos-version-min -> target=macabi?
Running ag shows us some clues:
```
ag ios --depth 0
webrtc.gni
30:if (is_ios) {
31:  import("//build/config/ios/rules.gni")
```

rules.gni is not really interesting, but `BUILD/config/ios/BUILD.gn` certainly is.

Here we can replace 
```
common_flags += [ "-mios-simulator-version-min=$ios_deployment_target" ]
```
with Catalyst's

```
--target=x86_64-apple-ios13.0-macabi
```

Next stop is sysroot, it's defined in `config/ios/ios_sdk.gni`
I've blatantly replaced generated values with constants to override them.

```
-  ios_sdk_path = ""
+  ios_sdk_path = "/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.15.sdk"

```

Trying to build with
```ninja -C out/ios_64 framework_objc```
gives us
```clang: error: using sysroot for 'MacOSX' but targeting 'iPhone' [-Werror,-Wincompatible-sysroot]```
At this moment  I was quite lost. The flags to clang look identical to what Xcode does, but magic doesn't happen.

If I found out, webrtc bundles it's own **llvm toolchain** (that's why it takes 16GB of storage?). Luckily, they still support Xcode's toolchain, which can be changed at 
```
BUILD/toolchain/toolchain.gni
-  use_xcode_clang = false
+  use_xcode_clang = true
```

With that, it properly compiles and works, 🎉.
I hope you enjoyed reading and acquired basic knowledge of how to approach it yourself.


### Thanks
To [siuying](https://twitter.com/siuying) for the suggestion to write this blog and technical ideas

To [cdnbye](https://github.com/cdnbye/WebRTCDatachannel), which was the starting point.
