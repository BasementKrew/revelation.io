---
layout: post
title:  "Swift Sub Strings"
date:   2014-11-06 10:00:00
author: "<a href='http://daltoniam.com'>Dalton Cherry</a>"
summary: "Swift Sub Strings"
tags: apple, ios, swift, http, strings, bumblebee
---

This week we jump into the interesting world of Swift sub strings. Creating sub strings in Swift is quite different then the familiar `NSRange` sub strings of our days of future past with Objective-C (yes that was an attempt at an X-Men reference). 

Let's start off with what we already know, Objective-C sub string use `NSString`

```objc
NSString *world = [@"Hello World" substringWithRange:NSMakeRange(6,5)];
//start at character 6 then move forward 5 characters to end up with "world"
```
Now here is our Swift equivalent: 

```swift
let text = "Hello World"
let world = text[advance(text.startIndex,6)..< text.endIndex]
```

Those coming scripting languages probably see a range like this as fairly normal, but let's break it down. We have a string named `text`. We can create a sub string by using the power of Swift ranges. The `advance` method takes an index and moves it by a certain amount, in the case 6. The `endIndex` points to "past the end" of the string. This means it doesn't return the last character of the string, but rather can be used for ranges like above instead of having to do something like `text[advance(text.startIndex,6)..<advance(text.startIndex,countElements(text))]`. Overall this saves us a few CPU cycles and looks nicer.

Now I am sure you are wondering, why can't I just use `Int` like in all the scripting languages? I believe this boils down to how the string index is designed. There is an `extension` on `String` in the Swift framework (basically the Swift standard library methods). This defines the structure `Index`, so publicly it would be known as `String.Index`. The `BidirectionalIndexType` protocol that `String.Index` implements allows the `String` structure to be iterated over. Likewise the `advance` method takes values that implement `ForwardIndexType` protocol which `BidirectionalIndexType` implements. This creates an orthogonal design by allowing the `advance` to be reused and the other containers in Swift to to use the same `BidirectionalIndexType` protocol. This underlining design principal makes a lot of sense, but it would be nice if a even higher level extension was added as well to allow the `Int` sub strings. You can implement this yourself with something as simple as:

```swift
extension String
{
    subscript(i: Int) -> Character {
        return self[advance(startIndex, i)]
    }
    
    subscript(range: Range<Int>) -> String {
        return self[advance(startIndex, range.startIndex)..<advance(startIndex, range.endIndex)]
    }
}
```

Which then would allow you to use `Int` ranges.

```swift
let text = "Hello World"
let world = text[6..< countElements(text)]
```


[Twitter](https://twitter.com/daltoniam)

[Swift Ranges](https://developer.apple.com/library/mac/documentation/Swift/Conceptual/Swift_Programming_Language/BasicOperators.html#//apple_ref/doc/uid/TP40014097-CH6-XID_125)