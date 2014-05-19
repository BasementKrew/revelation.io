---
layout: post
title:  "Adventure in Animation"
date:   2014-05-19 10:00:00
author: "<a href='http://daltoniam.com'>Dalton Cherry</a>"
summary: "This week, we cover the unsung hero of UI, Animations."
tags: Objective-C, objc, animation, OpenGL, ES, graphics, Core, Animation
---

Animations. The power that makes our applications feel real and dynamic. The closest thing many developers will ever come to programming games, since every developer wanted to program a game at some point in their lives (don't lie you know it is true). You have certainly seen animations and might have even used some in your own programming, but what technically is an animation? An animation is defined as: "the process of creating a continuous motion and shape change illusion by means of the rapid display of a sequence of static images that minimally differ from each other". That sounds very official, to put it simply, it means changing something slightly and rapidly, until it appears as it changed to its new state.

Now that we know what the technical definition of an animation is, let's do a real quick historical background. The concept of animation as a whole has pretty much been around as long as people have. Painting and drawings through history have tried to communicate the idea of animations, but it was not until the last century with the introduction of film and computers, have animations truly been the dynamic ones we know today. Computer animation has its roots in film, as the industry wanted a way to bring imagination to life. As computers improved, animations techniques did as well, creating the powerful animations frameworks and game engines we use today, and there appears to be no slowing down of this trend. With the introduction of iOS 7, Apple now has a game and animation engine baked right into their SDK. The signs can't be ignored, we must become the master of animations!

Now that you have successfully been persuaded into become a master of animations, let dig down into my personal favorite animation (and arguably one of the simplest) framework, Core Animation.

![](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/CoreAnimation_guide/Art/ca_architecture_2x.png)

In a nutshell, Core Animation builds on top of a lot of powerful graphic libraries to provide an easy to use API. For those less akin to graphics, OpenGL is the defacto standard of open source graphics libraries and runs at the heart of most major OSes (most meaning, not Windows). I will provide a link to their site at the end of the article if you want to take a look (you know you should). Alright example time!

```objc
UIView *testView = [UIView alloc] initWithFrame:CGRectMake(0,20,60,60)];
testView.backgroundColor = [UIColor redColor];
 [UIView animateWithDuration:0.25 animations:^{
        CGRect frame = testView.frame;
        frame.origin.x = 100;
        testView.frame = frame;
    } completion:^(BOOL f){
      //do something after the animation is finished (if needed).
    }];
```

That is fairly straight forward. Most standard applications exist in a 2D coordinate space (x,y). This makes sense as most of the applications we interact with are in a flat, 2D space of our monitor. Some coordinate systems are done with 0,0 at the lower left corner of the view, while others do 0,0 at the top left corner of the view. The UIView coordinate system is the done with the top left system.

Now `testView` has a beginning `x` of 0. This means in relation to its super view, it is 0 pixels away from the edge of it (meaning it touches the very left edge). The same applies for the y coordinate from the top of the super view. All these values together mean that `testView` is 0 pixels away from the left edge, 20 pixels away from the top edge, 60 pixels wide, and 60 pixels tall. Now the fun part, `UIView animateWithDuration:animations:` is a wonderfully simple API provided by Apple to perform simple animations. The first value is the length of the whole animation. The second value is a block that the animated changes are performed in. The change above means in 0.25 (1/4) second we will move the view's `x` coordinate from 0 to 100. With some simple math we can calculate that 100 divide by 0.25 is moving the `x` value by 4 pixels every 0.01 seconds. Now we don't know the exact rate that the drawing intervals happen at (we could though!), but we can conclude that it will appears though the view moved from 0 to 100 in a smooth fashion.  The example above is fairly simple, but demonstrations the overall concept and goal of animations, to give the illusion of movement.

Now we have covered, the definition, history, and an example of animations, but why would you use them? Never fear, another example is on the way from my own personal experience (yea! for personal tie-ins). First off, let's start with how my view looked.

![](/assets/images/post_images/instee_fte_static.png)

Not bad, if I do say so myself. What could animations do to improve that? After a long, hard, look, we felt it was not immediately obvious that the plus arrows needed to be tapped. We need a way to draw attention and communicate the this was an action, which caused us to do this:

![](/assets/images/post_images/instee_fte_pulse.gif)

We are now drawn to tap one of the boxes, just by simply making them do a pulse animation. Nothing in the overall UI had to change, but the addition of an animation did wonders for communicating what action the user needed to perform. Our job as programmers is to tastefully use these animations to better communicate to our users and create a more delightful experience.

I will close this article by stating I am by no means a graphics or animation expert. I still have lots to learn and the depth of graphics and animation knowledge to learn is both staggering and exciting. My hope is that by seeing a bit of history, a simple and real world example, you will start your quest to further your animation knowledge and create better applications for your users. The application I showcased above is my current project of Instee. You can check it out here: [Instee](http://insteeapp.com/). Also if you want some helpful examples of different iOS animation you can check out my animation library in the link below. As always, feedback, random rants, questions, and comments are appreciated. [@daltoniam](http://www.twitter.com/daltoniam)

- [OpenGL](http://www.opengl.org/)

- [Core Animation](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/CoreAnimation_guide/Introduction/Introduction.html)

- [Instee](http://insteeapp.com/)

- [DCAnimationKit](https://github.com/daltoniam/DCAnimationKit)


