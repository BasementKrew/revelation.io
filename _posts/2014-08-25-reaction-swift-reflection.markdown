---
layout: post
title:  "Reaction to Swift Reflection"
date:   2014-08-25 10:00:00
author: "<a href='http://daltoniam.com'>Dalton Cherry</a>"
author_image: "http://www.gravatar.com/avatar/2fdc7b889f35118a7334187b15c5b957.png?r=x&amp;s=320"
summary: "This week we explore reflection in Swift through JSON parsing"
tags: Objective-C, objc, apple, osx, ios, swift, json, parsing, reflection
---

JSON parsing in Swift. Arguably the most talked about and debated Swift topic to date. None of these articles have really explored parsing JSON with reflection, similar to [http://golang.org/pkg/encoding/json/#Unmarshal](http://golang.org/pkg/encoding/json/#Unmarshal) (we are pretty big Go fans around here). First off, let's start with a quick Swift reflection primer.

```swift
let str = "some string"
var mirror = reflect(str) //reflect takes AnyObject
//mirror contains different values of the string
println("mirror inspect: \(mirror.value)")
println("mirror inspect type: \(mirror.valueType)")
println("mirror count: \(mirror.count)") //This is the count of properties of that object
```

The `reflect` method creates a `mirror` object which is basically a copy of the values representing that object. `value` is the actual value of the object and `valueType` is the class. It is important to note that when the object is a optional (e.g. `var str: String?`) it's `valueType` and `value` are `nil`. This makes sense as the object does not yet have a value assigned to it. Once a value is assigned, it behaves the same as a non optional, other than being tagged as a optional type in the `mirror` disposition property. The last notable property is the `count` which is the count of the properties of the object. This can be used to know how many objects to loop through to get the properties as mirror objects as well. With a little bit of recursion, one can easily inspect all the values of a object.

The `reflect` method can be super handy for inspecting things at runtime for debugging purposes, but I was really looking for a way to convert raw JSON into usable objects. For me, this is where the faster and less dynamic side of Swift really started to show. As far as I can tell, there was no way to set a value to an arbitrary object like in Objective-C or Go. Objective-C has `setValue:forKey:` and Go has `Set` to allow a property's value to be set from a string based key.

 Here is an Objective-C example. This uses KVC and I recommend checking out the docs [here](https://developer.apple.com/library/mac/documentation/cocoa/reference/foundation/Protocols/NSKeyValueCoding_Protocol/Reference/Reference.html#//apple_ref/occ/instm/NSObject/setValue:forKey:).

```objc
//A simple example object
@interface TestObject : NSObject

@property(nonatomic, copy)NSString *name;
@property(nonatomic, strong)NSNumber *age;

@end

//some KVC wonderfulness
TestObject *object = [[TestObject alloc] init];
//we set the property's value based on it's name
[object setValue:@"Dalton" forKey:@"name"];
[object setValue:@(23) forKey:@"age"];
```

We could always convert to Objective-C containers in Swift to get this to work, but I was looking for a way without a large amount of conversion. This same reflection task is accomplished in Go and Rob Pike has written a great article with some nice examples: [http://blog.golang.org/laws-of-reflection](http://blog.golang.org/laws-of-reflection). Also the docs for Go reflection is [here](http://golang.org/pkg/reflect/).

 I haven't found a way to accomplish this in Swift. I assume this is due to the fact Swift is closer to C++ design paradigms, which also has a lack of JSON to object libraries or code examples. Since Swift is still in beta, my hope is that a dynamic assignment method comes out to make JSON to object serialization possible. If anyone has found a way to make this possible, please don't hesitate to share. As always, feedback, comments, and random rants are appreciated.

[Twitter](https://twitter.com/daltoniam)
