---
title:  "Numbers and Swift"
date:   2015-01-13T10:00:00-07:00
author: "<a href='http://daltoniam.com'>Dalton Cherry</a>"
author_image: "http://www.gravatar.com/avatar/2fdc7b889f35118a7334187b15c5b957.png?r=x&amp;s=320"
tags: swift, scalar, type, int, double, char, number
---

2 + 2 = 4. Aren't numbers grand? They are the basic building blocks for all data we work with and Swift is no exception. Speaking (or typing) of numbers, I was working on [SwiftLog](https://github.com/daltoniam/SwiftLog) the other day and came across a `NSFileManager` call that returned a `NSNumber` in Objective-C and a `UInt64` in Swift. The different types inspired me to write this article. Objective-C only has `NSNumber` for a number object, while Swift has a whole list:

`Int8`, `Int16`, `Int32`, `Int64`, `UInt8`, `UInt16`, `UInt32`, `UInt64`, `Float`, `Double`, `Float80`. `Int` maps to `Int32` or `Int64` depending on the platform architect. The same holds true for `UInt`. This makes sense though, as Swift is aiming for safety and speed versus the simple design of `NSNumber`. Let's break down an example.

Let's say you have an `Int` and an `Int64`. These 2 can't be compared as they are different types. Since the `Int` is smaller than the `Int64` let's convert it to `Int64`.

```swift
let i = 1 //this is an Int
let big: Int64 = 10000000 //this is an Int64

//create a Int64 version of i
let bigI = Int64(i)
if big == bigI {
  //do stuff since the compare now works
}
```

Simple and straight forward. This is also safer than `NSNumber` as you can't accidentally use `intValue` and truncate the larger number. Swift protects us from ever making this mistake by having the different types. These looks familiar to the C scalar types used in Objective-C which is convenient as we should have a pretty good clue to which each value maps to. It is important to note, that even though these look a lot like the C scalar types, they are in fact objects (`structs`). Apple provides a different set of types for direct C equivalence. A list can be found in the Swift programming manual. You can find a link to that at the end of the article.

Since we now know Swift's numeric types are `structs`, this also shows the orthogonal design of Swift. Just like the `Array` and `String` type, most of the functionally is apart of hierarchy of protocols. This protocol based design allows the same `Comparable` protocol to be used for all the types. This way something as simple as an `Int` all the way to an `Array` can implement the same protocol. We can even have our own types or data structures implement the same protocols and enforce the same, expected behavior. Pretty slick and orthogonal (just love saying the word!).

I believe an important takeaway here is seeing the different design patterns of Swift versus Objective-C. Swift favors speed and safety by reducing the number of errors that can be made. Objective-C on the other hand was designed to be simple. I don't think either one is necessarily wrong or right, but they certainly do showcase the evolution of programming. As always questions, comments, feedback, discussions, and random rants are appreciated. Find me at [@daltoniam](http://twitter.com/daltoniam)

- [Swift Numeric Types](https://developer.apple.com/library/ios/documentation/General/Reference/SwiftStandardLibraryReference/NumericTypes.html)

- [Swift C Types](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/BuildingCocoaApps/InteractingWithCAPIs.html#//apple_ref/doc/uid/TP40014216-CH8-XID_11)

- [Boolean Protocol](https://developer.apple.com/swift/blog/?id=8)