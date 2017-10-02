---
layout: post
title:  "Cool Codable Coding"
date:   2017-10-02 12:30:00
categories: 'dalton'
tags: Swift
keywords: swift codable 4 four other article json joy jsonjoy cool coding
cover: 'assets/images/codable-cover.png'
navigation: True
---


We are back! Wow has it really been two years? Ironically our last post was celebrating the one year mark and new things to come and then radio silence... 

Alas, we missed you all and we are back with a fresh new design and content. This week I wanted to cover Swift 4's `Codable` protocol. I know, I can hear your groans now about another `Codable` article, but I wanted to use this article to showcase `Codable` in a practical HTTP API setting. Plus it gives me the excuse to show off the new version 3 of [SwiftHTTP](https://github.com/daltoniam/SwiftHTTP) and to offically deprecate [JSONJoy](https://github.com/daltoniam/JSONJoy-Swift) (Apple decided to create even more joy).

First off, if you want a great technical explanation of the `Codable` protocol, please check out Mike Ash's wonderful [post](https://www.mikeash.com/pyblog/friday-qa-2017-07-14-swiftcodable.html) about it. I am going to focus on a more high level and example driven implementation, so without further ramblings, example time!


 Let's write a quick web server in Go (because we like Go and it gives us a full example to work with).

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
		w.Write([]byte("{\"status\": \"ok\"}"))
	})

	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

That code is fairly straight forward. Any request it gets it send back some JSON that says "OK". Now onto the Swift!

We create a struct that implements the `Codable` protocol.

```swift
struct Response: Codable {
    let status: String
}
```

That's it! Most default cases this will handle our JSON needs and it should be able to create our model off those properties. Now let's hook this up to our SwiftHTTP web request.

```swift
HTTP.GET("http://localhost:8080/bar") { response in
    if let error = response.error {
        print("got an error: \(error)")
        return
    }
    do {
        let decoder = JSONDecoder()
        let resp = try decoder.decode(Response.self, from: response.data)
        print("completed: \(resp.status)")
    } catch let error {
        print("decode json error: \(error)")
    }
}
``` 
Now I'm sure you have questions about the edge cases on JSON parsing with this simple example. I know you are thinking: "what about handling JSON's snake case vs Swift's camel case"? "How do I handle arrays"? "What about side loaded models?" "how about dates"? I said this article was practical did I not? We got examples for all those cases. Stacks of examples. Ok, only four of them, but enjoy them none the less!

### Casing

Now let's say we have some JSON that has the typical snake case.

```javascript
{
    "id" : 1,
    "first_name": "John",
    "last_name": "Smith",
    "age": 25
}

```

Here is our corresponding Swift model that uses camel case.

```swift
struct User: Codable {
    let id: Int
    let firstName: String
    let lastName: String
    let age: Int
}
```

These properties unfortunately don't map since their names are different. We could name just change our Swift (or JSON) properties to match, but that looks ugly and feels akward when everything else is camel case in Swift. We can easily fix that by changing our Swift model to include:

```swift
struct User: Codable {
    let id: Int
    let firstName: String
    let lastName: String
    let age: Int
    private enum CodingKeys: String, CodingKey {
        case id
        case firstName = "first_name"
        case lastName = "last_name"
        case age
    }
}
```

That takes care of that. Now we can keep our code looking consistent and pretty. The main cavet here is we have to include all of our properties names in the `CodingKeys` enum even if we only need to change one property mapping. 

### Arrays

Now parsing Array's is very straight forward. Take the JSON from this API: https://jsonplaceholder.typicode.com/photos. If you are to lazy to click or tap that link here is an example:

```javascript
[
  {
    "albumId": 1,
    "id": 1,
    "title": "accusamus beatae ad facilis cum similique qui sunt",
    "url": "http://placehold.it/600/92c952",
    "thumbnailUrl": "http://placehold.it/150/92c952"
  },
  {
    "albumId": 1,
    "id": 2,
    "title": "reprehenderit est deserunt velit ipsam",
    "url": "http://placehold.it/600/771796",
    "thumbnailUrl": "http://placehold.it/150/771796"
  },
  /*...... 4998 more photos in the array.... */
```

The model:

```swift
struct Photo: Codable {
    let albumId: Int
    let id: Int
    let title: String
    let url: String
    let thumbnailUrl: String
}
```


```swift
HTTP.GET("https://jsonplaceholder.typicode.com/photos") { (response) in
    if let error = response.error {
        print("got an error: \(error)")
        return
    }
    let decoder = JSONDecoder()
    do {
        let photos = try decoder.decode([Photo].self, from: response.data) //notice the array brackets [] around the Photo model
        for photo in photos {
            print("photo id: \(photo.id) url: \(photo.url)")
        }
    } catch let error {
        print("got an error: \(error)")
    }
}
``` 
Easy right? You throw some brackets in there and are good to go.

### Date

I included this one because I was able to take it directly from the WWDC slides. I regret nothing.

```javascript
{
  "name": "Monalisa Octocat",
  "email": "support@github.com",
  "date": "2011-04-14T16:00:49Z"
}
```

Swift model:

```swift
struct Author : Codable {
  let name: String
  let email: String
  let date: Date
}
```


```swift
HTTP.GET("http://localhost:8080/author") { response in
    if let error = response.error {
        print("got an error: \(error)")
        return
    }
    do {
        let decoder = JSONDecoder()
        decoder.dateDecodingStrategy = .iso8601 //It will properly map the JSON string to the Date object since it knows the format
        let author = try decoder.decode(Author.self, from: response.data)
        print("date: \(author.date)")
    } catch let error {
        print("decode json error: \(error)")
    }
}
``` 

The default for `dateDecodingStrategy` is `deferredToDate`. The [docs](https://developer.apple.com/documentation/foundation/jsonencoder.dateencodingstrategy/2895359-deferredtodate) says this means: `The strategy that uses formatting from the Date structure`. Not super helpful in our case without having a `Date` structure yet, but at least we can easily change it to a fairly standard `iso8601` with a simple property change.

### Side loaded models

Now we bring it all together with a side loaded model. 

```javascript
{
	"status": "success",
	"users": [{
		"id": 1,
		"first_name": "Dalton",
		"last_name": "Cherry"
	}, {
		"id": 2,
		"first_name": "Austin",
		"last_name": "Cherry"
	}],
	"user": {
		"id": 3,
		"first_name": "John",
		"last_name": "Doe"
	}
}
```

The models:

```swift
struct User: Codable {
    let id: Int
    let firstName: String
    let lastName: String
    private enum CodingKeys: String, CodingKey {
        case firstName = "first_name"
        case lastName = "last_name"
        case age
    }
}

struct Response: Codable {
    let status: String
    let users: [User]
    let user: User
}
```

Now for the request:

```swift
HTTP.GET("http://localhost:8080/bar") { response in
    if let error = response.error {
        print("got an error: \(error)")
        return
    }
    do {
        let decoder = JSONDecoder()
        let resp = try decoder.decode(Response.self, from: response.data)
        print("completed: \(resp.status)")
        for user in resp.users {
            print("id: \(user.id) name: \(user.firstName) \(user.lastName) age: \(user.age)")
        }
        print("single user: \(resp.user.id) name: \(resp.user.firstName) \(resp.user.lastName) age: \(resp.user.age)")
        print("completed: \(resp.status)")
    } catch let error {
        print("decode json error: \(error)")
    }
}
``` 
Fairly anti climax since it is the last example, but it does show case how easy it is to implement `Codable`. 

### End

I am quite excited about the `Codable` protocol so far and it does eliminate quite a bit of the standard boilerplate with JSON parsing in Swift. I am also excited to get back to blogging and we are going to try and keep the momentum going. Austin has some fun Go things coming up next to keep the content flowing with the hope for a bi-weekly schedule of posting, so [follow](https://twitter.com/vluxeio) the Vluxe twitter account to keep in the loop (or in youtube terms... SHARE, LIKE, AND SUBSCRIBE). We will talk (write?) to y'all soon.  

- [Mike Ash's technical post](https://www.mikeash.com/pyblog/friday-qa-2017-07-14-swiftcodable.html)
- [Apple's encoding doc](https://developer.apple.com/documentation/foundation/archives_and_serialization/encoding_and_decoding_custom_types)
- [WWDC Video](https://developer.apple.com/videos/play/wwdc2017/212/)
- [SwiftHTTP](https://github.com/daltoniam/SwiftHTTP)

