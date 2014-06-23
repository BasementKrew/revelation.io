---
layout: post
title:  "Stylish Swift"
date:   2014-06-23 10:00:00
author: "<a href='http://daltoniam.com'>Dalton Cherry</a>"
summary: "Now that swift is a few weeks old, we jump into some tips and tricks to maximize your swift style."
tags: Objective-C, objc, swift, apple, cocoa, tips, tricks, best, practice, help, style
---

Stylish code is something we all should strive for. It is a topic that is constantly debated and is heavily influenced by personal preference. It also varies between languages on what is considered stylish. Swift is still young and changing, so its style is still developing. I will attempt to correlate some of the language's features from it's influences to hopefully show some style that differs from Objective-C. Code is always the best example, so let's jump right in.

**Tuples**

Tuples are probably the most significance change to our method style. Tuples allow a method to have multiple return types. This can replace the need to pass in several parameters as pointers as we commonly do in C or Objective-C. Here is a common way to create Objective-C method:

```objc
-(BOOL)doWork:(NSString*)someParam error:(NSError**)error
```

We would use this method like so:

```objc
NSError *error = nil;
BOOL status = [self doWork:@"someString" error:&error];
if(error) {
  NSLog("We got an error");
  return;
}
if(status) {
  NSLog("success");
} else {
  NSLog(@"failure");
}
```

As with most languages that are influenced by C, methods can only return one type in Objective-C. Tuples fix this in swift.

```swift
func doWork(someParam: string) -> (status: bool, error: NSError?)
```

The method usage would look like so:

```swift
let response = doWork("someString")
if response.error {
  println("we got an error")
  return
}
if response.status {
  println("success")
} else {
  println("failure")
}
```

We can see this can come in handy when dealing with several types in one method. Now I bet your are probably thinking, "what if I just need to update an already existing variable? Your tuples can't fix that!!!!". Very observant Dr. Watson, well done! Luckily for us, Swift has a way to handle this, with the `inout` keyword. This allows a variable to be pass by reference and work the same as the tuple free days.


**Convenience Methods**

It is common to see convenience methods in Objective-C simplify object initialize (or commonly known as factory methods). Here is a common convenience from cocoa touch: 

```objc
[UIButton buttonWithType:UIButtonTypeCustom];
```

This creates a new UIButton without having to call the normal `alloc` and `init` paradigm. Swift's `init` method and pretty syntax eliminates the need to create methods like this. Here is an example of this from my own swift library ([SwiftHTTP](https://github.com/daltoniam/SwiftHTTP) for those interested).

The method started off following the original Objective-C pattern:
```swift
class HTTPPair: NSObject {
        var value: AnyObject!
        var key: String!
        
        init() {
        }

        class func pair(value: AnyObject, key: String?) -> HTTPPair {
          var pair = HTTPPair()
          pair.value = value
          pair.key = key
          return pair
        }
    }

```
Code complies and works fine, but when it came to using it, it looked something like this:

```swift
collect.append(HTTPPair.Pair(object, key: key))
```
Everything worked, but wasn't as stylish as it could be. Also we where forcing `value` to by optional instead of explicit. The improved version:

```swift
class HTTPPair: NSObject {
    var value: AnyObject
    var key: String!
    
    init(value: AnyObject, key: String?) {
        self.value = value
        self.key = key
    }
  }
```

Then it would be used like so:

```swift
collect.append(HTTPPair(value: object, key: key))
```

That is much more stylish. We got to make value be explicit to reduce the amount of variable state and even got to shorten the code a bit. There is also a convenience keyword in swift that can be used for `init()`. Quick example below:

```swift
class Food {
    var name: String
    init(name: String) {
        self.name = name
    }
    convenience init() {
        self.init(name: "[Unnamed]")
    }
}
```

This convenience style in swift makes for a simpler initialize and saves us some code (the best code you write is the code you don't!). 

**Filter/Map/Reduce**

Swift includes a lot of functional programming ability. This allows for some pretty powerful expressions that would normally take much more code. I will once again refer to [SwiftHTTP](https://github.com/daltoniam/SwiftHTTP) for the compare and contrast.

```objc
+(NSString*)queryStringFromParametersWithEncoding:(NSDictionary *)parameters encoding:(NSStringEncoding)stringEncoding
{
    if(!parameters)
        return nil;
    NSMutableArray *mutablePairs = [NSMutableArray array];
    for(QueryStringPair *pair in [[self class] queryStringPairsFromDictionary:parameters]) {
        [mutablePairs addObject:[pair URLEncodedStringValueWithEncoding:stringEncoding]];
    }
    return [mutablePairs componentsJoinedByString:@"&"];
}
```

This method basically encodes the HTTP parameters with the `parameters` dictionary into a string to send with the HTTP request. Overall this code is pretty straight forward, but Swift can even make it more stylish:

```swift
func stringFromParameters(parameters: Dictionary<String,AnyObject>) -> String {
        return join("&", map(serializeObject(parameters, key: nil), {(pair) in
            return pair.stringValue()
            }))
    }
```

Using the commonly functional methods of `join` and `map`, Swift allows us to turn this block of code into a one line method. This of course is a very simple example of the functional power within Swift, but one can quickly see the benefits on the lines of code saved. I would recommend looking into some of common use case of `filter/map/reduce` in other languages if you are interested in leveraging this methods to their full capacity.

**SubString**

This section is a bit of a source spot for Swift. Swift is shaping up to be a really powerful and great language, but it is incomplete in several aspects. One of these aspects I came across is creating substrings. This is just really difficult and seems to be incomplete. Luckily some some folks are already on this case. This [radar](http://openradar.appspot.com/radar?id=6373877630369792) has the code:

```swift
extension String {
    
    subscript (idx: Int) -> String
        {
        get
        {
            return self.substringWithRange(
                Range( start: advance( self.startIndex, idx),
                    end: advance( self.startIndex, idx + 1 )  )
            )
        }
    }
    
    subscript (r: Range<Int>) -> String
        {
        get
        {
            return self.substringWithRange(
                Range( start: advance( self.startIndex, r.startIndex),
                    end: advance( self.startIndex, r.endIndex + 1 ))              )
        }
    }
    
    func substringFrom(start: Int, to: Int) -> String
    {
        return (self.substringFromIndex(start)).substringToIndex(to - start + 1)
    }

    //handy example of CoreFoundation bridge in Swift
    func escapeStr() -> String {
        var raw: NSString = self
        var str = CFURLCreateStringByAddingPercentEscapes(kCFAllocatorDefault,raw,"[].",":/?&=;+!@#$()',*",CFStringConvertNSStringEncodingToEncoding(NSUTF8StringEncoding))
        return str
    }
}
```

This String extension allows for handy expressions like so:

```swift
let str = "hello world!"
println("the third char: \(str[3])") //prints l
let sub = str[0..5] //sub now equals "hello"
let safeStr = str.escapeStr() //makes the string URL safe "hello%20world"
```

Hopefully this will be add into core of Swift soon, but until then we have a fairly simple extension to assist with common string needs.

Swift is a great language and I have appreciated the differences in style from the well worn path of Objective-C. As time goes on, hopefully a strong and pretty style firms up, so we can keep writing the best and most readable code possible. As always, questions, comments, rants, and random dialog are appreciated.


- [@daltoniam](https://twitter.com/daltoniam)

- [Swift book](https://itunes.apple.com/us/book/swift-programming-language/id881256329?mt=11)

- [SwiftHTTP](https://github.com/daltoniam/SwiftHTTP)

