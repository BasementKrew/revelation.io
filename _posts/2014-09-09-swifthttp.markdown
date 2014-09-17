---
layout: post
title:  "SwiftHTTP"
date:   2014-09-11 10:00:00
author: "<a href='http://daltoniam.com'>Dalton Cherry</a>"
summary: "The thoughts and workings of the SwiftHTTP library."
tags: apple, osx, ios, swift, http, wwdc, http, swifthttp
---

This morning I was reflecting on how much has changed in the last 90 days for us as Apple devs. The release of Swift, new UI for OS X, new APIs for both iOS and OS X, and not to mention the release of the Apple Watch. The breakneck pace of the change and innovation is almost overwhelming. This reflection reminded me of my time at WWDC and learning Swift by writing [SwiftHTTP](https://github.com/daltoniam/SwiftHTTP). The library has certainly change a lot since that first version but still has the same goal, learn Swift and simplify HTTP requests.

Let's start with the most basic use case, a GET request (basically a HTTP library's "hello world").

This is a simple server in Go, just to demonstrate both sides of the request.

```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func main() {
    http.HandleFunc("/bar", func(w http.ResponseWriter, r *http.Request) {
        log.Println("got a web request")
        fmt.Println("header: ", r.Header.Get("someKey"))
        w.Write([]byte("{\"status\": \"ok\"}"))
    })

    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

Now for the HTTPTask.

```swift
var request = HTTPTask()
request.requestSerializer = HTTPRequestSerializer()
request.requestSerializer.headers["someKey"] = "SomeValue"
request.responseSerializer = JSONResponseSerializer()
request.GET("http://localhost:8080/bar", parameters: ["param": "param1", "array": ["first array element","second","third"], "num": 23], success: {(response: HTTPResponse) -> Void in
    if (response.responseObject != nil) {
        println("got response: \(response.responseObject!)")
    }
    }, failure: {(error: NSError) -> Void in
        println("got an error: \(error)")
})
```

A lot to cover in the small amount of code, so let's jump in.

**HTTP Verbs**

SwiftHTTP supports all common HTTP verbs such as GET, POST, PUT, DELETE, etc. The `enum` of the verbs can be found in `HTTPTask.swift`.

```swift
public enum HTTPMethod: String {
    case GET = "GET"
    case POST = "POST"
    case PUT = "PUT"
    case HEAD = "HEAD"
    case DELETE = "DELETE"
}
```

There also built in convenience methods with the verb names that they preform. See `Operation Queues` below for more information.

**requestSerializer**
The requestSerializer is responsible for all the settings of a HTTP request. This includes request headers, how parameters are serialized, and standard NSURLRequest settings like `cachePolicy` and so forth. All requestSerializers are a subclass of `HTTPRequestSerializer` and is the default if one isn't specified. Here are some common examples of how to use a requestSerializer:

Set a header:

```swift
var request = HTTPTask()
request.requestSerializer.headers["someKey"] = "SomeValue"
```

Serialize parameters:

```swift
var request = HTTPTask()
request.POST("http://domain.com/create", parameters: ["param": "hi", "something": "else", "key": "value"], success: {(response: HTTPResponse) -> Void in

    },failure: {(error: NSError) -> Void in

    })
```

The default `HTTPRequestSerializer` serializes the parameters according to the HTTP spec of [query string](http://en.wikipedia.org/wiki/Query_string). The above output would look like:

```
param=hi&something=else&key=value
```

This fully supports Arrays, Dictionaries, Strings, and Ints. There is also an included `JSONRequestSerializer` that can be used if the parameters need to be embed in the HTTP body as JSON. The output would look like so:

```
{"key":"value","something":"else","param":"hi"}
```

Also an important note is that if a variable is used for the parameters, the type must be explicitly set.

```swift
//WRONG, will not build
//let params = ["param": "param1", "array": ["first array element","second","third"], "num": 23, "dict": ["someKey": "someVal"]]

//we have to add the explicit type, else the wrong type is inferred 
let params: Dictionary<String,AnyObject> = ["param": "param1", "array": ["first array element","second","third"], "num": 23, "dict": ["someKey": "someVal"]]

var request = HTTPTask()
request.POST("http://domain.com/create", parameters: params, success: {(response: HTTPResponse) -> Void in

    },failure: {(error: NSError) -> Void in

    })
```

This is an issue with the Swift type inference inferring the wrong type, so explicitly setting is required.

**responseSerializer**
The responseSerializer is responsible for object serialization that is returned in the HTTP response as `responseObject`. In practice, this means we can make a request return a serialized object instead of just returning a `NSData` object. SwiftHTTP includes `JSONResponseSerializer` that will serialize the response into a `Foundation` object using `NSJSONSerialization`.

```swift
var request = HTTPTask()
request.responseSerializer = JSONResponseSerializer()
request.GET("http://localhost:8080/bar", parameters: nil, success: {(response: HTTPResponse) -> Void in
    if (response.responseObject != nil) {
        //response.responseObject is now a Dictionary of the JSON instead of just raw NSData
        println("got response: \(response.responseObject!)")
    }
    }, failure: {(error: NSError) -> Void in
        println("got an error: \(error)")
})
```

In addition to the `JSONResponseSerializer` custom subclasses can be created for `responseSerializer`. Some good examples of this could be an image or XML serializer. If a `responseSerializer` isn't specified, then a `NSData` of the response is returned.

**HTTPResponse**

The `HTTPResponse` objects represents the values of an HTTP response. This includes the `statusCode`, `mimeType`, and `responseObject`. The `responseObject` is of type `AnyObject` so a `responseSerializer` can convert this into the proper object representation of the data.

**Files**

To upload a file with a request, simply use the `HTTPUpload` object. This object takes either a file url or an `NSData` blob and mimeType. The request is then sent as a [multipart/form-data](https://www.ietf.org/rfc/rfc1867.txt) request.

```swift
let fileUrl = NSURL.fileURLWithPath("/Users/dalton/Desktop/file")
var request = HTTPTask()
request.POST("http://domain.com/1/upload", parameters:  ["param": "hi", "something": "else", "key": "value","file": HTTPUpload(fileUrl: fileUrl)], success: {(response: HTTPResponse) -> Void in

    },failure: {(error: NSError) -> Void in

    })
```

**Operation Queues**

One of the most powerful features of SwiftHTTP is the `NSOperation` support. Every `HTTPTask` creates `HTTPOperations`, which makes working with `NSOperationQueue` very simple.

```swift
//create an queue.
let operationQueue = NSOperationQueue()
operationQueue.maxConcurrentOperationCount = 2

//create an HTTPTask
var request = HTTPTask()
//using the create method, an HTTPOperation is returned.
var opt = request.create("http://vluxe.io", method: .GET, parameters: nil, success: {(response: HTTPResponse) -> Void in
    if response.responseObject != nil {
        let data = response.responseObject as NSData
        let str = NSString(data: data, encoding: NSUTF8StringEncoding)
        println("response: \(str)") //prints the HTML of the page
    }
    },failure: {(error: NSError) -> Void in
        println("error: \(error)")
})
//To start the request, we simply add it to our queue
if opt != nil {
    operationQueue.addOperation(opt!)
}
```

The example shows how easy it is to use an `NSOperationQueue` with SwiftHTTP. It is important to note that the convenience methods (GET,POST,PUT,etc) all call the create method shown and start the operation all on their own as seen below.

```swift
//pulled this directly from the HTTPTask class.
public func GET(url: String, parameters: Dictionary<String,AnyObject>?, success:((HTTPResponse) -> Void)!, failure:((NSError) -> Void)!) {
        var opt = self.create(url, method:.GET, parameters: parameters,success,failure)
        if opt != nil {
            opt!.start()
        }
    }
```

Again these methods are designed to be for convenience, so to work with your own operation queues, you need to use the `create` method.

**BaseURL and APIs**

SwiftHTTP can also be used to simplify API interaction. To do this, simply set the `baseURL` property and reuse the same HTTPTask object. All request will append the `baseURL` to the url requested.

```swift
var request = HTTPTask()
request.baseURL = "http://api.someserver.com/1"
request.GET("/users", parameters: ["key": "value"], success: {(response: HTTPResponse) -> Void in
    println("Got data from http://api.someserver.com/1/users")
    },failure: {(error: NSError) -> Void in

    })

request.POST("/users", parameters: ["key": "updatedVale"], success: {(response: HTTPResponse) -> Void in
    println("Got data from http://api.someserver.com/1/users")
    },failure: {(error: NSError) -> Void in

    })

request.GET("/resources", parameters: ["key": "value"], success: {(response: HTTPResponse) -> Void in
    println("Got data from http://api.someserver.com/1/resources")
    },failure: {(error: NSError) -> Void in

    })
```

This could also be combined with the operation queue functionally to provided queued API interaction (so orthogonal!).

**Closing**

The rapidly changing landscape of Cocoa development is certainly an exciting one. The opportunity to learn, try, and develop new things has always been a major reason I love programming and the release of Swift 1.0 is no exception. That all being said, this article will be a "living" article as changes and improvements are made to SwiftHTTP. As always, comments, thoughts, kudos, and random rants are appreciated.

[Twitter](https://twitter.com/daltoniam)

[SwiftHTTP](https://twitter.com/daltoniamhttps://github.com/daltoniam/SwiftHTTP)




