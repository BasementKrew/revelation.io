---
layout: post
title:  "Swift, Labels, Default Parameters, oh my!"
date:   2015-02-16 10:00:00
author: "<a href='http://daltoniam.com'>Dalton Cherry</a>"
author_image: "http://www.gravatar.com/avatar/2fdc7b889f35118a7334187b15c5b957.png?r=x&amp;s=320"
tags: swift, label, parameter, argument, method, function, default
---


The other day I was working on [Jazz](https://github.com/daltoniam/Jazz) and got to fully appreciate the convenience of default parameters in Swift. Default parameters combined with labels gives a simple and flexible method signature. This is in stark contrast to some other method signatures in other development environments. methods in these environments can have an upwards of 10 parameters, all with arbitrary names. Let's take a look at an example of such methods:

```c
LPTSTR szCmdline[] = _tcsdup(TEXT("\"C:\\Program Files\\MyApp\" -L -S"));
CreateProcess(NULL, szCmdline, /*...*/);
```

This is an old C++ Windows API (the shock and horror). Want to guess how many parameters this method has? 2, 3, 7? This method has a total of 10 parameters. Here is the full method signature:

```
  _In_opt_     LPCTSTR lpApplicationName,
  _Inout_opt_  LPTSTR lpCommandLine,
  _In_opt_     LPSECURITY_ATTRIBUTES lpProcessAttributes,
  _In_opt_     LPSECURITY_ATTRIBUTES lpThreadAttributes,
  _In_         BOOL bInheritHandles,
  _In_         DWORD dwCreationFlags,
  _In_opt_     LPVOID lpEnvironment,
  _In_opt_     LPCTSTR lpCurrentDirectory,
  _In_         LPSTARTUPINFO lpStartupInfo,
  _Out_        LPPROCESS_INFORMATION lpProcessInformation
);

```

Now in its defense, C++ does allow for default parameters, thus making the total _required_ parameters 4. The ambiguity arises when trying to figure out what value is being passed for which parameter. Swift solves this problem by pulling in the verbosity of Objective-C by allowing parameters to be labeled. Here is an example from Jazz:

```swift
public func play(length: NSTimeInterval, delay: NSTimeInterval = 0, springDamping: CGFloat = 1, velocity: CGFloat = 1, animation:((Void) -> Void)) -> Jazz
```

This allows for the method to be simple in average use cases, but still being flexible and clear. Here is the simplest use case:

```swift
play(0.25, animation: {
    //do animations
})
```

Now let's add some delay.

```swift
play(0.25, delay: 1.0, animation: {
    //do animations
})
```

Still very clear and we get to use the same method. Let's remove the delay and change the springDamping.

```swift
play(0.25, springDamping: 0.5, animation: {
    //do animations
})
```

We know exactly which parameter we are modifying and don't have to waste time checking the documentation or method signature. Now let's set both delay and springDamping.

```swift
play(0.25, delay: 1.0, springDamping: 0.5, animation: {
    //do animations
})
```

Simple, flexible, and clear. Everything we could want from a method. It is great to see how Swift pulled in some nice features from different languages and executed them in such a way that makes the features more powerful by being combined. As always questions, comments, feedback, discussions, and random rants are appreciated. Find me at [@daltoniam](http://twitter.com/daltoniam)

- [Swift Guide: Methods](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Methods.html)


