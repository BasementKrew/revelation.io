---
layout: post
title:  "Indirection with idiomatic interfaces"
date:   2014-04-14 08:00:00
author: "<a href='http://austincherry.me'>Austin Cherry</a>"
summary: "Interfaces. Simply a lanaguage ideal or a powerful use of indirection? In this article we will explore the uses of interfaces, how different languages use them. We will also see an example of golang uses them."
tags: interfaces, indirection, golang, Go, Java, ruby, Objective-C
---

Indirection with idiomatic interfaces. Ok maybe a little over doing it, but I need a couple of cool "i" words I could string together with "interfaces". On a more definitity note, interfaces do actually provide a nice layer of indirection or what most programmers would probably call abstraction. For those newer to computer science, those terms can seem a little vague so let's define them:

*indirection:* `indirectness or lack of straightforwardness in action, speech, or progression.`

*abstraction:* `the process of considering something independently of its associations, attributes, or concrete accompaniments`

Good, but still a little vauge. I think the best way I could describe it in a computer science terms, would be saying if some is abstract or indirect, we are trying to design a system that is flexible, reusable and extendable. All wonderful ideals when it comes to designing great software. Interfaces are a construct to provide us with just that. Since talk is cheap and code is what us problem solvers crave, I would like to provide some examples of interfaces in different languages. In my oh so humble opinion I believe the best place to start is with C.

```C
//cool C code here
```

Now I know what you are thinking. C doesn't have interfaces and that code is just a function pointer! To that I would say you are correct! I believe C is important to include as it is the base off which pretty much what all software is built. The upside to the function pointer is that it is providing some nice indirection and is quite fast. Why it is fast is something we will cover in another article. On the downside it is not really as flexible as we might like and has some not so pretty syntax. This leads us to our next contender, Java!

```Java
// some cool Java interface code here...
```

Excellent. This is providing us some nice flexiblity, with a pretty easy syntax.