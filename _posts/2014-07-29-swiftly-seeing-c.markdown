---
layout: post
title:  "Swiftly Seeing C"
date:   2014-07-29 10:00:00
author: "<a href='http://daltoniam.com'>Dalton Cherry</a>"
summary: "This week we dig into some C and Core Foundation tips in Swift."
tags: Objective-C, objc, apple, cocoa, websocket, osx, ios, swift, C, Core Foundation
---

After last week's break (vacations are fun!), we pick right back up on our Swift adventures. Last time we covered [websockets](working-with-websockets.html) and the latest addition to the open source family, [jetfire](https://github.com/acmacalister/jetfire). Now that the Objective-C version is finished, I felt the best way to learn about C and Core Foundation in Swift was to port [jetfire](https://github.com/acmacalister/jetfire). This new Swift version is appropriately named [starscream](https://github.com/daltoniam/starscream). Alright enough self promotion, onward to the examples!


**Bytes and NSData**

A common aspect of working with sockets is the manipulation of raw socket data or the bytes of `NSData`. Swift and Objective-C vary a bit on how you interact with the underlining bytes of a `NSData` object. Objective-C uses its C roots and resorts to a simple typecast to work with the bytes by returning the bytes as a `void` pointer. Swift on the other hand, does not like such loose programming and requires a bit more finesse to make this work.

Objective-C:

```objc
NSData *data = [NSData data]; //for the sake of an example, pretend some data is in this...
uint8_t *buffer = (uint8_t*)[data bytes];
//do stuff with buffer...
```

Swift:

```swift
let data = NSData() //for the sake of an example, pretend some data is in this...
let bytes = UnsafePointer<UInt8>(data.bytes)
//do stuff with bytes...
```

That wasn't so bad. The biggest difference is the need to be explicit in the usage of the bytes. Since `void` pointers and C cast don't exist in Swift, the use of `UnsafePointer` is how C bytes buffers are accessed in Swift.

**Core Foundation Bridging**

Core Foundation methods are an interesting bit in Swift. I believe that the plan would be to completely abstract the need to interact with these method directly in the long run, but many frameworks are still very much C frameworks (with good reason) and still required to get more custom work done. With that in mind, there are still ways to interact with these methods in Swift, but the bridging isn't quite as smooth as in Objective-C. Examples incoming:

Objective-C:

```objc
CFURLRef url = CFURLCreateWithString(kCFAllocatorDefault, (CFStringRef)self.url.absoluteString, NULL);
CFStringRef requestMethod = CFSTR("GET");
CFHTTPMessageRef urlRequest = CFHTTPMessageCreateRequest(kCFAllocatorDefault,
                                                         requestMethod,
                                                         url,
                                                         kCFHTTPVersion1_1);
CFHTTPMessageSetHeaderFieldValue(urlRequest,
                                 (__bridge CFStringRef)headerWSUpgradeName,
                                 (__bridge CFStringRef)headerWSUpgradeValue);
CFHTTPMessageSetHeaderFieldValue(urlRequest,
                                 (__bridge CFStringRef)headerWSConnectionName,
                                 (__bridge CFStringRef)headerWSConnectionValue);
CFHTTPMessageSetHeaderFieldValue(urlRequest,
                                 (__bridge CFStringRef)headerWSProtocolName,
                                 (__bridge CFStringRef)headerWSProtocolValue);
CFHTTPMessageSetHeaderFieldValue(urlRequest,
                                 (__bridge CFStringRef)headerWSVersionName,
                                 (__bridge CFStringRef)headerWSVersionValue);
CFHTTPMessageSetHeaderFieldValue(urlRequest,
                                 (__bridge CFStringRef)headerWSKeyName,
                                 (__bridge CFStringRef)[self generateWebSocketKey]);
CFHTTPMessageSetHeaderFieldValue(urlRequest,
                                 (__bridge CFStringRef)headerOriginName,
                                 (__bridge CFStringRef)self.url.absoluteString);
CFHTTPMessageSetHeaderFieldValue(urlRequest,
                                 (__bridge CFStringRef)headerWSHostName,
                                 (__bridge CFStringRef)[NSString stringWithFormat:@"%@:%@",self.url.host,self.url.port]);

NSData *serializedRequest = (__bridge NSData *)(CFHTTPMessageCopySerializedMessage(urlRequest));
```

Swift:

```swift
func createHTTPRequest() {

    let str: NSString = _url.absoluteString
    let url = CFURLCreateWithString(kCFAllocatorDefault, str, nil)
    let urlRequest = CFHTTPMessageCreateRequest(kCFAllocatorDefault, "GET",
        url, kCFHTTPVersion1_1)

    self.addHeader(urlRequest, key: headerWSUpgradeName, val: headerWSUpgradeValue)
    self.addHeader(urlRequest, key: headerWSConnectionName, val: headerWSConnectionValue)
    self.addHeader(urlRequest, key: headerWSProtocolName, val: headerWSProtocolValue)
    self.addHeader(urlRequest, key: headerWSVersionName, val: headerWSVersionValue)
    self.addHeader(urlRequest, key: headerWSKeyName, val: self.generateWebSocketKey())
    self.addHeader(urlRequest, key: headerOriginName, val: _url.absoluteString)
    self.addHeader(urlRequest, key: headerWSHostName, val: "\(_url.host):\(_url.port)")

    let serializedRequest: NSData = CFHTTPMessageCopySerializedMessage(urlRequest.takeUnretainedValue()).takeUnretainedValue()
    self.initStreamsWithData(serializedRequest)
}
//Add a header to the CFHTTPMessage by using the NSString bridges to CFString
func addHeader(urlRequest: Unmanaged<CFHTTPMessage>,key: String, val: String) {
    let nsKey: NSString = key
    let nsVal: NSString = val
    CFHTTPMessageSetHeaderFieldValue(urlRequest.takeUnretainedValue(),
        nsKey,
        nsVal)
}
```

As one can see, the bridging isn't quite as straight forward. Currently Swift `String` objects don't bridge straight to `CFString` objects. This seems strange as `NSString` objects can bridge to `CFString` and `String` can bridge to `NSString`, but not directly to a `CFString`. Swift requires a intermediate `NSString` object to bridge to a `CFString` object in order to work properly. I hope this changes in the future, as it would be nice to go straight to Core Foundation objects from the Swift equivalents (if possible).

This is also the case with other Core Foundation objects such as the `CFData` to `NSData` or `CFDictionaryRef` to `NSDictonary` and `Dictionary`. The bridging works by using the `takeUnretainedValue` method which as far as I can tell, is the same as doing a toll-free bridge in Objective-C. This returns the underlining memory of the object, which can be used to properly bridge to the Core Foundation method.

**Single Character Constants**

This is a bit arbitrary , but I think it is worth sharing. Swift does not allow single `char` literals. By that I mean you can't do the `'a'` or `'\n'` anymore. The only way I found this can be accomplished is like so: `UInt8("a")` or `UInt8("\n")`. Not the end of the world, but it threw me at first as that is yet another common C expression I didn't realized didn't make the jump to Swift.

**The End**

There is certainly more to cover with Core Foundation and Swift and some of this may change in the next few releases before Xcode 6 becomes Golden Master, but my hope is this article obliterates some the time investment in starting with C and Core Foundation in Swift. Not to continue on a cycle of self promotion, but I am not aware of any other libraries that that are in both Swift and Objective-C to compare and contrast, so I could recommend you take a look at [jetfire](https://github.com/acmacalister/jetfire) and [starscream](https://github.com/daltoniam/starscream). They are good way to see (or what I have figured out so far) the transition of Core Foundation from Objective-C to Swift. Any other libraries that also demonstrates using C and Core Foundation, please don't hesitate to share (nobody likes a hoarder!!!). As always, questions, comments, feedback, and random rants are appreciated.

[Twitter](https://twitter.com/daltoniam)

[jetfire](https://github.com/acmacalister/jetfire)

[starscream](https://github.com/daltoniam/starscream)

[Swift Docs](https://developer.apple.com/library/prerelease/ios/documentation/swift/conceptual/buildingcocoaapps/InteractingWithCAPIs.html)

