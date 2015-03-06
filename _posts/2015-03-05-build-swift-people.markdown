---
layout: post
title:  "Swift People"
date:   2015-03-06 10:00:00
author: "<a href='http://daltoniam.com'>Dalton Cherry</a>"
author_image: "http://www.gravatar.com/avatar/2fdc7b889f35118a7334187b15c5b957.png?r=x&amp;s=320"
tags: apple, ios, swift, http, swifthttp, skeets, image, remote, app, build, tableview, cocoa, pods, storyboard, JSONJoy
---

The last few weeks, I have been asked the same question, "how do I load asynchronous images or data in a UITableView in Swift?". While I love to help out, answering the same question can get a bit repetitive. I then used my powers of deduction and said "wait, I have a blog! I should write an article on this.". A few hours later Swift People was born.

We will create the "Swift People" project, which will walk through the basics of creating a new project with framework management and load remote data and images into a tableView. Let's start with the framework management to get [Skeets](https://github.com/daltoniam/Skeets), [SwiftHTTP](https://github.com/daltoniam/SwiftHTTP), and [JSONJoy](https://github.com/daltoniam/JSONJoy-Swift) in our project. The full project is available [here](https://github.com/Vluxe/SwiftPeople).

## Framework Management

There are several great options for framework management, namely: [Carthage](https://github.com/Carthage/Carthage), [Rogue](https://github.com/acmacalister/Rogue) and [CocoaPods](http://cocoapods.org). I would recommend checking out each to see which one best fits in your workflow. Swift People will use CocoaPods since it is the most well known.

First off I would like to add a quick disclaimer about using CocoaPods. As of today (March 6th, 2015) CocoaPods Swift framework support is in **beta**. You will have to install the pre-release in order to get started, so install and use with **discretion** and at **your own risk**.

`gem install cocoapods --pre`

If you haven't install CocoaPods yet, check out the install docs [here](http://cocoapods.org).

Next create a Xcode project (I named mine SwiftPeople). Then from terminal navigate to your new project directory and run:

 ```
 touch Podfile
 ```

 Now open the `Podfile` file with your favorite text editor and add these lines:

 ```
platform :ios, '8.0'
pod 'SwiftHTTP', :git => "https://github.com/daltoniam/SwiftHTTP.git"
pod 'Skeets', :git => "https://github.com/daltoniam/Skeets.git"
pod 'JSONJoy-Swift', :git => "https://github.com/daltoniam/JSONJoy-Swift.git"
 ```

 Next run this in terminal:

```
pod install
```

This will install the pods. Close your Xcode project and open the newly created workspace file. This can be done from terminal if you want.

```
open SwiftPeople.xcworkspace
```

## API

Now that we have a workspace, Next we need to get some mock data to build our app with. I came across [Random User](https://randomuser.me). This handy API provides all the mock user data we need to build a feed of people. Let's make a SwiftHTTP request and build a JSONJoy model of that data.

```swift
static func getUsers(finished:((Array<User>) -> Void)) {
    let task = HTTPTask()
    task.responseSerializer = JSONResponseSerializer()
    task.GET("http://api.randomuser.me", parameters: ["results": "30"], success: { (response: HTTPResponse) in
        if let data = response.responseObject as? NSData {
            var collect = Array<User>()
            //get a decoder of the data
            let decoder = JSONDecoder(data)
            //check the array is valid and loop through the array
            if let arr = decoder["results"].array {
                for subDecoder in arr {
                    //flatten the json
                    let user = subDecoder["user"]
                    if user.error == nil {
                        collect.append(User(user))
                    }
                }
            }
            //move to the main thread as we are done with our processing
            dispatch_async(dispatch_get_main_queue(), {
              finished(collect)
            })
        }
        }, failure: { (error: NSError, response: HTTPResponse?) -> Void in
        println("failed to get random users: \(error)")
    })
}
```

This method is static, so we can call it without having to create a new `User` object. I placed this my `viewDidLoad` of my view controller for this example, which we will see in the next section. Now let's take a look at the JSON handling in the `User` object.

```swift
struct User: JSONJoy {
    let name: Name
    let picture: Picture
    //computed property to return a full name: "John Doe".
    var displayName: String { return "\(name.first) \(name.last)" }

    //implement the JSONJoy protocol
    init(_ decoder: JSONDecoder) {
        name = Name(decoder["name"])
        picture = Picture(decoder["picture"])
    }
}

//represents the User's name
struct Name: JSONJoy {
    let first: String
    let last: String

    //JSONJoy init method
    init(_ decoder: JSONDecoder) {
        first = safeString(decoder,"first")
        last = safeString(decoder,"last")
    }

    //fetch the random users from the network
    static func getUsers(finished:((Array<User>) -> Void)) {
      SwiftHTTP request and such from above....
    }
}

//represents the User's picture
struct Picture: JSONJoy {
    let thumbnail: String
    let small: String
    let medium: String
    let large: String

    //JSONJoy init method
    init(_ decoder: JSONDecoder) {
        thumbnail = safeString(decoder,"thumbnail")
        small = safeString(decoder,"small")
        medium = safeString(decoder,"medium")
        large = safeString(decoder,"large")
    }
}

//unwrap and read a string safely
func safeString(decoder: JSONDecoder, key: String) -> String {
    if let str = decoder[key].string {
        return str
    }
    return ""
}
```

That takes care of the JSON processing and should be fairly straight forward.

## TableView

Now that we have an API that can retrieve data and transform it into models, let's hook it up the the UITableView. There is a storyboard in the sample project if you want to see where the tableView comes from. Let's fetch the users and load them into our tableView.

```swift
override func viewDidLoad() {
    super.viewDidLoad()
    //our static method from the User model
    User.getUsers{ (users: Array<User>) in
        self.users = users //hold onto the user models
        self.tableView.reloadData()
    }
}
```

Now let's implement the tableView datasource.

```swift
//number of rows
func tableView(tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
    if let users = self.users {
        return users.count
    }
    return 0
}

//style the cells
func tableView(tableView: UITableView, cellForRowAtIndexPath indexPath: NSIndexPath) -> UITableViewCell {
    var cell: UserTableViewCell! = tableView.dequeueReusableCellWithIdentifier("user") as? UserTableViewCell
    if cell == nil {
        cell = UserTableViewCell(style: .Default, reuseIdentifier: "user")
    }
    //we have a cell, now let's style it.
    if let users = self.users {
        if indexPath.row < users.count {
            let user = users[indexPath.row]
            //we have our user model, let's update the cell with the data
            cell.imgView.image = nil
            cell.txtLabel.text = user.displayName
            //if the cell is being recycled, make sure to cancel the old url it was loading.
            if cell.currentImgUrl != "" {
                ImageManager.cancel(cell.currentImgUrl)
            }
            //fetch the remote image using skeets
            ImageManager.fetch(user.picture.medium, progress: { (progress: Double) in
                }, success: { (data: NSData) in
                    //got the image, load it into the UI
                    cell.imgView.image = UIImage(data: data)
                    cell.currentImgUrl = ""
                }, failure: { (error: NSError) in
                    println("failed to get image: \(error)")
            })
            cell.currentImgUrl = user.picture.medium
        }
    }
    return cell
}
```

## Fin

That's it! Here is what our what our final product will look like.

![](/assets/images/swiftpeople.png)


This is a simple example, but gives a good start point to start building apps that interacts with asynchronous data. As always, questions, comments, general thoughts, and random rants are appreciated. [@daltoniam](https://twitter.com/daltoniam)


- [Swift People](https://github.com/Vluxe/SwiftPeople)
- [randomuser.me](https://randomuser.me)
- [Skeets](https://github.com/daltoniam/Skeets)
- [SwiftHTTP](https://github.com/daltoniam/SwiftHTTP)
- [JSONJoy](https://github.com/daltoniam/JSONJoy-Swift)



