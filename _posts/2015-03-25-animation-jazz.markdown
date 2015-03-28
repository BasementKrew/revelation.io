---
layout: post
title:  "Animation Jazz"
date:   2015-03-25 10:00:00
author: "<a href='http://daltoniam.com'>Dalton Cherry</a>"
author_image: "http://www.gravatar.com/avatar/2fdc7b889f35118a7334187b15c5b957.png?r=x&amp;s=320"
tags: Jazz, animation, ios, swift, pretty, ui, ux, designer, login, invision, mockup, view, uikit, images, reproduction
---

I was recently given an idea by a former colleague to find and implement cool dribbble animations for iOS. I was sharing the work I had done in [Jazz](https://github.com/daltoniam/Jazz) and it seemed like a good way to create examples for the library. The animation I choose inspired me to completely re-purpose [Jazz](https://github.com/daltoniam/Jazz) into a layer based component library.

![](https://d13yacurqjgara.cloudfront.net/users/62319/screenshots/1945593/shot.gif)

This animation was created for Secret Project at invision by [Anton Aheichanka](https://dribbble.com/madebyanton). Now I will admit my reproduction isn't detail for detail of the mockup, but for a few hours of work I am happy with the result.

![](/assets/images/login.gif)

You can find the full code [here](https://github.com/Vluxe/AnimationSeries/tree/master/login). Let's break down each component starting with the obvious ones. The background is a `UIImageView`. I just found a stock image and added the `UIImageView` as the first subview to the view hierarchy. The two `UITextField` are the same as the `UIImageView` (minus the image of course). The interesting part is our custom button and loading dialog, so let's move on to that.

These custom controls are built to respond to animated changes and included in Jazz. The first thing we do is add the button to the view hierarchy, just like the `UIImageView` or `UITextField`. We also need to add the loading dialog to the view, but hide it, as we aren't loading anything yet.

```swift
let offset: CGFloat = 50

//layout the button and add it to the view
let pad: CGFloat = 15
let h: CGFloat = 60
button.frame = CGRectMake(pad, self.view.frame.size.height-(h+pad+offset), self.view.frame.size.width-(pad*2), h)
button.corners = UIRectCorner.AllCorners //all the corners are rounded
button.autoresizingMask = .FlexibleHeight | .FlexibleWidth
button.color = UIColor(red: 253/255.0, green: 56/255.0, blue: 105/255.0, alpha: 1)
button.highlightColor = UIColor(white: 0.0, alpha: 0.3)
button.ripple = true //adds the ripple tap button effect
button.textLabel.text = NSLocalizedString("Sign In", comment: "")
button.textLabel.textColor = UIColor.whiteColor()
button.cornerRadius = h/2 //round the corners
self.view.addSubview(button)

//handle the button tap
button.didTap{
    println("apple workaround, button tapped") //I believe this is fixed in swift 1.2
    //Using Jazz, animate the button changes and show the loading dialog
    Jazz(0.25, animations: {
        self.button.frame = CGRectMake((self.button.frame.size.width-h)/2, self.button.frame.origin.y, h, h)
        self.button.textLabel.text = ""
        self.button.cornerRadius = h/2
        self.button.enabled = false
        self.loadingView.hidden = false
        let inset: CGFloat = 30
        self.loadingView.frame = CGRectMake(self.button.frame.origin.x+((self.button.frame.size.width-inset)/2),
            self.button.frame.origin.y+((self.button.frame.size.height-inset)/2), inset, inset)
        self.loadingView.start(speed: 0.5)
        return [self.button,self.loadingView]
    })
    //pretend to send a request for the login
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT,0), {
        sleep(1) //fake wait time here
        //now back to the main thread to do some more drawing
        dispatch_async(dispatch_get_main_queue(), {
            self.view.bringSubviewToFront(self.button)
            self.loadingView.stop()
            //expand the button view to fill the screen and open new view controller
            Jazz(0.25, animations: {
                Jazz.expandView(self.button, scale: 12)
                Jazz.moveView(self.button, x: -(self.button.frame.size.width/4), y: -40)
                return [self.button]
            }).play(0.25, animations: {
                self.button.alpha = 0
                if let app = UIApplication.sharedApplication().delegate as? AppDelegate {
                    if let window = app.window {
                        window.rootViewController = EndController(nibName: nil, bundle: nil)
                    }
                }
                return [self.button]
            })
        })
    })
}

//add the loading dialog to the view. Hide it since it isn't used at first
loadingView.color = UIColor.whiteColor()
loadingView.lineWidth = 2 //how thick is the line of the loading dialog?
loadingView.autoresizingMask = .FlexibleHeight | .FlexibleWidth
loadingView.hidden = true
self.view.addSubview(loadingView)
```

The button and loading dialog are added to the view hierarchy, then the animations are scheduled to be run once the button is tapped. This causes the loading dialog to be displayed. The cool part is the way the animations are chained to run one after another. This makes expressing complex animations easy. Jazz also makes creating custom controls easy, by being protocol based and having a base `Shape` class that allows most basic shapes to be easily animated. I don't want to do to much self promotion here, so I suggest checking out the learning more about playing animation Jazz from from the [docs](https://github.com/daltoniam/Jazz).

In conclusion, I now ask you for help. I would love to be able to post these reproduction animations to dribbble, but alas, I am still a prospect. If you have an invite and would like to share, I would much appreciate it. If you are a designer or just found a cool animation and would like to have the work featured, let me know. I would like this could become a recurring series of different animations and UIs. As always, questions, comments, and random rants are appreciated. [@daltoniam](https://twitter.com/daltoniam)

- [Designer](https://dribbble.com/madebyanton)

- [Dribble Project](https://dribbble.com/shots/1945593-Login-Home-Screen)

- [Swift Project](https://github.com/Vluxe/AnimationSeries/tree/master/login)

- [Jazz](https://github.com/daltoniam/Jazz)

