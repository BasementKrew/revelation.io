---
layout: post
title:  "SwiftHTTP"
date:   2014-09-09 10:00:00
author: "<a href='http://daltoniam.com'>Dalton Cherry</a>"
summary: "The thoughts and workings of the SwiftHTTP library."
tags: apple, osx, ios, swift, http, wwdc
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
The requestSerializer is responsible for all the settings of the request. This includes request headers, how parameters are serialized, and standard NSURLRequest settings like `cachePolicy`. All requestSerializers are a subclass of `HTTPRequestSerializer` and if a requestSerializer isn't specified, `HTTPRequestSerializer` is used. Here is some common examples:

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
It also supports the multi form spec for file upload.

This fully supports Arrays, Dictionaries, Strings, and Ints. There is also an included `JSONRequestSerializer` that can be used if the parameters need to be embed in the HTTP body as JSON. The output would look like so:

```
{"key":"value","something":"else","param":"hi"}
```

**responseSerializer**
The responseSerializer is responsible for object that is returned for `responseObject`. This means we can make a request return a serialized object instead of just returning a `NSData` object. SwiftHTTP includes `JSONResponseSerializer` that will serialize the response into a `Foundationn` object using `NSJSONSerialization`.

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
Custom subclasses can be created for `responseSerializer`. Examples of this could be an image serializer or an XML one. If a `responseSerializer` isn't specified, then a `NSData` of the response is returned. 

**HTTPResponse**

This is the  

**File Upload/Download**


**Operation Queues**

**BaseURL and APIs**

**Closing**







