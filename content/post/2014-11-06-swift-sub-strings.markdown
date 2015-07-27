---
title:  "Swift Substrings"
date:   2014-11-06T10:00:00-07:00
author: "<a href='http://daltoniam.com'>Dalton Cherry</a>"
author_image: "http://www.gravatar.com/avatar/2fdc7b889f35118a7334187b15c5b957.png?r=x&amp;s=320"
summary: "Swift Substrings"
tags: apple, ios, swift, string, sub, osx, extension, index, range
---

This week we jump into the interesting world of Swift substrings. Creating substrings in Swift is quite different then the familiar `NSRange` substrings of the days of future past with Objective-C (yes, that was an attempt at an X-Men reference).

Let's start off with what we already know, Objective-C substring with `NSString`.

```objc
NSString *world = [@"Hello World" substringWithRange:NSMakeRange(6,5)];
//start at character 6 then move forward 5 characters to end up with "world"
```
Now here is our Swift equivalent:

```swift
let text = "Hello World"
let world = text[advance(text.startIndex,6)..< text.endIndex]
```

Those comfortable with scripting languages probably see a range like this as fairly normal, but let's break it down. We have a string named `text`. We can create a substring by using the power of Swift ranges. The `advance` method takes an index and moves it by a certain amount, in this case 6. The `endIndex` points to "past the end" of the string. This means it doesn't return the last character of the string, but rather can be used for ranges like above instead of having to do something like:

```swift
text[advance(text.startIndex,6)..< advance(text.startIndex,countElements(text))]
```

Overall this saves us a few CPU cycles and looks nicer.

Now I am sure you are wondering, why can't I just use `Int` like in all the scripting languages? I believe this boils down to how the string index is designed. There is an `extension` on `String` in the Swift framework (basically the Swift standard library methods). This defines a structure `Index`, so publicly it would be known as `String.Index`. The `BidirectionalIndexType` protocol that `String.Index` implements allows the `String` structure to be iterated over. Likewise the `advance` method takes values that implement `ForwardIndexType` protocol which `BidirectionalIndexType` implements. This creates an orthogonal design by allowing the `advance` method to be reused and the other containers in Swift to to use the same `BidirectionalIndexType` protocol. This underlining design principal makes a lot of sense, but some might argue that it would be nicer if an even higher level extension was added as well to allow the `Int` substrings. You can implement this yourself with something as simple as:

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

This of course has the trade off of a being a bit slower, which is my suspicious on why it is has not been included in the Swift standard library (or is it framework?!?). Overall substrings in Swift benefit from a more orthogonal design and I appreciate the thoughtfulness that has gone into the creation of the Swift. As always, questions, comments, feedback, discussions, and random rants are appreciated.


[Twitter](https://twitter.com/daltoniam)

[Swift Ranges](https://developer.apple.com/library/mac/documentation/Swift/Conceptual/Swift_Programming_Language/BasicOperators.html#//apple_ref/doc/uid/TP40014097-CH6-XID_125)