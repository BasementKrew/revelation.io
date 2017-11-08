---
layout: post
title:  "Figuring Out Frame Layouts in Swift"
date:   2017-11-08 12:30:00
categories: 'dalton'
tags: Swift
keywords: swift 4 four frame layout autolayout auto subview view
cover: 'assets/images/frame-layout-cover.jpg'
navigation: True
---


Confession time. I forgot to write an article last week as promised, but I think this one is worth the wait. Today we are back to talk about some frame layouts. We are going back to the days before Auto Layout and showing you how to make a subview with nothing more than your will, wit, and a bit of math. Let’s layout!

The first step is to create a view subclass to work with. Here is a simple example:

```swift
class TestView: UIView {
    let titleLabel = UILabel()
    let detailLabel = UILabel()
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        setup()
    }
    
    required init?(coder aDecoder: NSCoder) {
        super.init(coder: aDecoder)
        setup()
    }
    
    func setup() {
        backgroundColor = UIColor.black
        
        titleLabel.textColor = UIColor.white
        titleLabel.numberOfLines = 1
        titleLabel.font = UIFont.boldSystemFont(ofSize: 17)
        addSubview(titleLabel)
        
        detailLabel.textColor = UIColor.white
        addSubview(detailLabel)
    }
    
    override func layoutSubviews() {
        super.layoutSubviews()
        //label frame layouts here....
    }
}
```

Now the first thing to notice is the `layoutSubviews` method. This is where we will do the layout for our subviews (like our labels). We add all of our subviews frame layouts inside this method. Here is that code:

```swift
override func layoutSubviews() {
    super.layoutSubviews()
    let pad: CGFloat = 5
    var top: CGFloat = pad
    titleLabel.frame = CGRect(x: pad, y: top, width: bounds.width - (pad * 2), height: 20)
    top += titleLabel.bounds.height + pad
    
    detailLabel.frame = CGRect(x: pad, y: top, width: bounds.width - (pad * 2), height: bounds.height - top)
}
```

Pretty simple, eh? (side note, can I use `eh` even though I'm not Canadian?). The first thing to note is on iOS the view coordinate system starts in the top left corner (0,0). This means adding any value to `x` or `y` will move the view down (y) or to the right (x). The width and height do exactly what you would expect and give the view its size. The call to `bounds` is asking the parent view (our subclass `TestView`) how big it is and laying out the subviews accordingly. We use pad to make the math easy and have even spacing on both the left and right side of the label. This is why the `bounds.width - (pad * 2)` has times 2 of `pad`. The view moved to the right 5 points to the right and thus we take 10 points from the width to account for it and add the same amount of spacing on the left. Now all we need to do is add `TestView` to our view hierarchy.

 ```swift
 //I add this code in my viewController's viewDidLoad
let testView = TestView(frame: CGRect(x: 20, y: 80, width: view.bounds.width - 40, height: 60))
testView.titleLabel.text = "Dalton Cherry"
testView.detailLabel.text = "I like using frame layouts."
view.addSubview(testView)
 ```
 
 ![](/assets/images/base-frame-view.png)

Now for the most powerful part of using frame layouts.... Animations! Frame layouts allow for a simpler system of animation then Auto Layout's constraints. Let's say we want to move the view around or change its size. We can directly update it in our animation closure like so:

```swift
//this code would exist after a button click or something
UIView.animate(withDuration: 0.25, animations: {
    testView.frame.size.height = 90
    testView.frame.origin.x = 0
    testView.backgroundColor = UIColor.red
})
```

 ![](/assets/images/frame-layout-first-animation.gif)

That is it. We don't have to mess around with updating constraints (unless we are mixing the two systems, more on that at the end). We just update the view and it does its thing. We can also animate the `transform` property to do simple rotation and expanding animations as seen here:

```swift
UIView.animate(withDuration: 0.25, animations: {
    testView.transform = testView.transform.scaledBy(x: 1.1, y: 1.1) //make view 10% bigger by scaling it up
})
```

 ![](/assets/images/frame-layout-second-animation.gif)


Now this is only the start in some of the things that can be accomplished with frame layouts. The main takeaway here is frame layouts can be a valuable tool when developing views that might have extensive animations or highly custom UI and/or UX. This of course comes with the trade off of having to manage your own layouts mathematically which might be more cognitive overhead when dealing with things like rotation, size classes, different layouts for different devices, etc. The other neat thing is you can actually mix both frame layout views and Auto Layout managed views with just a few properties. You can find more info about that in Apple's WWDC talk [here](https://developer.apple.com/videos/play/wwdc2015/219/) but just a quick TLDR; it centers around the `translatesAutoresizingMaskIntoConstraints` property. The documentation for that can be found [here](https://developer.apple.com/documentation/uikit/uiview/1622572-translatesautoresizingmaskintoco).

I hope this brief primer was informative on some of the uses and trade offs of frame layouts. I plan to follow this article up with a few more to showcase more complex animations and even maybe some of the more tedious code that AutoLayout might simplify in non-animated cases. As always, any questions, concerns, random rants, praise, or thoughful disagreements send it my way.

- [Apple's view property](https://developer.apple.com/documentation/uikit/uiview/1622572-translatesautoresizingmaskintoco)
- [WWDC Video](https://developer.apple.com/videos/play/wwdc2015/219/)
