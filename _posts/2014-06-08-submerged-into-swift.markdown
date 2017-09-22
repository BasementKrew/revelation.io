---
layout: post
title:  "Submerged into Swift"
date:   2014-06-08 10:00:00
categories: 'dalton'
summary: "Now that WWDC, has finished up, we get back in the swing of things with a in depth look at swift."
tags: Swift
keywords: Objective-C objc swift wwdc missed week apple cocoa
---

Now with WWDC in our rear view mirror (always a great conference to attend), we turn our focus to exploring one of tge biggest announcements of the conference, Swift. Swift is Apple's new language that is "Objective-C without the C". When I first heard this, I was a bit shaken. It is no secret that I am a pretty big fan of C and Objective-C and this announcement almost felt like a betrayal of those core languages. Ok, that might be a touch dramatic, but Swift was going to take some time for me to adjust to. I felt it was only fair to give the language a full try and compare and contrast the differences with Objective-C and see how it fairs for its first release. I started digging through the Swift source files and created a simple HTTP library to give myself a solid Swift base with APIs I was already familiar with, before I casted my judgmental C loving eyes upon it. Here is what I found:

**Assignment and Syntax:**

This is an obvious one, but it is worth mentioning, as it changes your coding syntax quite a bit. Let's take a quick example:

Objective-C version:

```objc
NSString *name = @"Dalton";
NSString *str = [NSString stringWithFormat:@"hello %@, how are you doing?",name];
NSArray *array = @[@"value1",@"value2",@"value3"];
NSMutableArray *collect = [NSMutableArray arrayWithCapacity:10];
for(int i = 0; i < 10; i++) {
    [collect addObject:[NSString stringWithFormat:@"index is: %d",i]];
}
```

Swift version:

```swift
var name = "Dalton"
//some string interpolation
var str = "hello \(name), how are you doing?"
//standard, NOT mutable array or like a NSArray
let array = ["value1","value2","value3"]

//a mutable array like NSMutableArray
var collect = Array<String>()
for var i = 0; i < 10; i++ {
    collect[i] = "index is: \(i)"
}
```

Not so bad. Swift certainly removes some of the verbose syntax and obliterates the need for the @ prefix to ensure C compatibility. The first thing we can pick out is no semicolons, which is slick and an expectation at this point for any new language. Also notice there is no more pointer syntax, so all the stars are gone. Swift uses `type inference` which means the variable type is inferred from the value that is assigned, so the variable `name` is known to be of the type of `String` because the value assigned to it is `"Dalton"`. Likewise, it knows that `array` is an Array because the value assigned to it is `[]`. Next, notice the keywords of `var` and `let` are used instead of the actual type (which works hand and hand with type inference). This is really powerful to express the mutability of the variable, without have to create a whole new class like its `NSArray` and `NSMutableArray` counterparts. `var` means the variable is mutable and `let` means the variable is not mutable, which will not change after its assignment. No need for a separate subclass, which is a pretty handy way of handling mutability with objects. The last thing to notice is the string interpolation (the `\()` in the string's ""). This is a fairly standard thing in most scripting languages today, and is a touch easier to write out and understand than the variadic arguments of old.

**Standard Library**

This is pretty subtle, yet important detail in using Swift. The standard library types like `String` and `Array` are NOT the same thing as `NSString` or `NSArray`. Swift is designed as a stand alone language from the Cocoa runtime, so these types don't directly relate. The kids at Apple didn't leave us hanging though, and created some automatic logic to properly bridge these types.

```swift
var str = "stuff,something,else,content,blah"
let array = str.componentsSeparatedByString(",");
for val in array {
    println("val is: \(val)")
}
```

Let's break this down. `str` is a standard Swift `String`, but `componentsSeparatedByString` is a `NSString` method. This works find though, because the smart folks at Apple made a bridge from Objective-C, so the `componentsSeparatedByString` returns an `Array` instead of a `NSArray`. This is super handy, but isn't perfect. There are some types that just can't bridge quite yet, like a `String` to a `CFString`.

```swift
//have to create an NSString object, then pass it to CFURLCreateStringByAddingPercentEscapes, as it expects a CFString, which only bridges to a NSString currently.
var raw: NSString = self
var str = CFURLCreateStringByAddingPercentEscapes(kCFAllocatorDefault,raw,"[].",":/?&=;+!@#$()',*",CFStringConvertNSStringEncodingToEncoding(NSUTF8StringEncoding))
```

As the comment outlines, we have to explicitly create a NSString object, instead of letting it be bridged. Swift can't handle this yet, but my hope is that this will be resolved in a future seed or have a pure Swift implementation. Also, to be clear:

```swift
var nsStr: NSString = "this is an NSString"
println("an NSString: \(nsStr)")
```

Adding a `:Type` will explicitly set the variable's type, in case the inferred type is not what you wanted.


**Generics**

```swift
var collect = Array<String>()
var dict = Dictionary<String,AnyObject>()
collect[0] = "hello"
collect[1] = "world"
dict["hello"] = "world"
dict["value"] = 1
for (key, object) in dict {
    println("key: \(key) object: \(object)")
}
for str in collect {
    println("str: \(str)")
}
```

Generics are a fairly common pattern in most languages, so I will not elaborate on them here, but if you are primarily a Objective-C programmer, this might seem a bit foreign. Basically you are setting the only type that can be used in the collection. In our example, this means the array `collect` can only hold string objects. The type is declare in the `<>` brackets, same as the Dictionary object. This may seem like we are losing functionally, but this makes a lot of sense in most contexts. Often when using a collection, we are only putting one type of object in it. Generics makes this clearer and "safer" so we don't cause ourself problems with the wrong type getting added. If we do need the same freedom Objective-C has, we can use the type of `AnyObject` which means we can put any object into the collection.

**Functions:**

Functions in Swift are much like functions in Go.

```swift
func doWork(parameters: Dictionary<String,AnyObject>, name: String) -> (String,String) {
//do stuff
}
```

Yeah, so this is way different then what we are use to, but it makes a lot of sense. We prefix every function with the `func` keyword, just like in Go. Next we declare a label of the argument, with the type after the label. In our example, `parameters` and `name` are both labels, to make working with the types that are getting passed in simple. The return type is a tuple of String objects which is the `(String,String)` thing after the ->. A cool part about Swift, much like Go, is that you can return multiple types from a function (tuple all the things!). This is especially useful, as we don't use pointers (there is the inout keyword, but we will save that for another article).

**Closures:**

Closures are pretty close to what Objective-C people know as blocks. There is some different syntax as it is a Swift function versus a block.

```swift
func run(parameters: Dictionary<String,AnyObject>!, success:((AnyObject?) -> Void)!, failure:((NSError) -> Void)!) {
//do stuff
}

That method gets called like so:
var task = HTTPTask(url: "http://google.com")
task.run(nil,{(response: AnyObject?) -> Void in
    println("success")
    },{(error: NSError) -> Void in
    println("failure")
    })
```

That is some closure joy right there. The second parameter declared `((AnyObject?) -> Void)!` is the success closure, which is the same as a block that would be called when the method completed successfully. This syntax means we expect AnyObject as the parameter type and the return type to be void. We call this method with some interesting syntax:

```swift
{(response: AnyObject?) -> Void in
  println("success")
}
```
Other than the `{}` wrapping it, the closure is a standard function declaration. After that, it uses the `in` keyword to have the body of the closure filled out. That is different syntax than I was use to seeing, but fairly straight forward once you wrap your head around it.

**Obscure Runtime Things:**

This section will be fairly brief, as I have not done all the research I should before writing an in-depth post. I have however, notice something in Swift I would like to share. Swift does not appear to use the same dynamic runtime as Objective-C. By this, I mean using reflection in Swift is not nearly as direct as Objective-C. I have had quite a bit of trouble trying to port [JSONJoy](https://github.com/daltoniam/JSONJoy) to Swift. The problem is, I don't see a way in the Swift runtime to reflect an object's properties. There is a `reflect()` method that exist in the runtime, but it does not appear to work for property inspection. I am going to go out on a limb and say this is because the Swift runtime does not boil everything down to `objc_msgsend` and instead uses the v_table directly, much like C++ or Go. I will not elaborate any further on that, as like I said, I have not done the research and I could be totally wrong (please correct me if I am!!). There just doesn't appear to be a simple way to do this level of reflection needed for JSONJoy to work. I will keep everyone apprised and I have opened a bug report with Apple to see if there is a way to make this happen.


**FIN**

In closing, it is important to note I didn't even get close to covering everything new in Swift. There is still enumerations, classes, structures, extensions, getters, setters, and a whole slew of new things. I really encourage anyone interested to read the Swift manual/book from Apple provide in the link below. It covers the language in fairly decent detail and is a great read to better understand Swift (make sure to write a bunch of Swift code too! Best way to learn is to do!).

The goal of this article was just to cover some of the big things I ran into in hopes that someone might benefit from my findings (and maybe me and the 10 other C/Objective-C loving programmers can start a support group or something). Nobody ever likes change, as it requires us to relearn new habits and we are naturally resistant, but Swift will certainly open the Cocoa platform to new programmers and brings some of the handy advances in new languages to Cocoa programming. Despite some of the quirks and possibly unwelcomed need to change, Swift is in-line with the ever striving goal to create better software. Swift is a young language and it will take some time to fully mature into the language that Objective-C is, but I am hopeful that this language will be a powerful tool in our pursuit of great software. As always, feedback, questions, comments, and random rants are appreciated. [@daltoniam](https://twitter.com/daltoniam).

- [Swift book](https://itunes.apple.com/us/book/swift-programming-language/id881256329?mt=11)


