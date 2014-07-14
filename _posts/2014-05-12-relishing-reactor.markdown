---
layout: post
title:  "Relishing the Reactor Pattern"
date:   2014-05-12 08:00:00
author: "<a href='http://austincherry.me'>Austin Cherry</a>"
summary: "To continue with our vein on concurrency, in today's article we are going to cover the reactor pattern powering some of the hottest open source libraries to date."
tags: concurrency, parallelism, asynchronous, C, reactor, node, libuv, eventmachine
---

For those not familiar with the "Reactor pattern", it is the design powering library and frameworks such as: Ruby's eventmachine, Python's Twisted, C's libev and probably the most notable, Node.js. Its massive jump in popularity over the last few years have is because of the C10K problem. For those not on the up and up with the C10K problem, it basically a review of different design patterns for web servers to handle web scale and support ten thousand clients simultaneously (hence the ten in C10K). This design pattern can actually find it's roots back in the 1995 in the book "Pattern Languages of Program Design" by Jim Coplien and Douglas C. Schmidt. So if this isn't a new idea, then why is there such a fuss in tech community about it? Turns out this pattern is really good at handling a lot of I/O calls without the overview and difficulty of a multi-threaded web server. No threads? The C programmer in me is having a bit of a panic attack, but fear not, let us discuss how this is possible.

![](http://img440.imageshack.us/img440/3262/reactordiagram.png)

The diagram above gives a pretty good idea of how this works. The selector notifies a connected client when there is an event of interest from the dispatcher, who is controlling all the I/O handlers. That's all great and dandy in theory, but how does that work in practice, with let's say... Node.js? (If you are interested in more of the theory of the Reactor pattern, I included a link at the bottom of the article for your reading pleasure.) Enter libuv. One of the C library that powers Node.js and most importantly, the library that provides the reactor pattern event loop we are examining. It was inspired from the libev library, which actually made up the Node.js internals until 0.4 when it was replaced by libuv. In practice libuv tends to use the higher performance method of kernel event notifications, such as epoll, kqueue, etc. There are a ton of articles that compare kqueue and epoll against the older and more generic select method of handling file descriptor events, so we won't belabor them (For those not familiar with file descriptors, I have a link at the end of the article to read more about. In an over simplification they are a way of managing I/O calls, such as file reading/writing, syscalls, HTTP requests, etc that the OS kernel normal is responsible for).

The advantage that these event driven systems provide over the older select system is easily explained with an example. Imagine being at your favorite restaurant. Would it be more efficient for everyone in the restaurant to go ask the cook if their food is ready yet, or wait for number to be called and go get it then? The same goes for select vs the other event-driven variants. In the select system you have to keep an array of file descriptors and loop over them every couple of seconds to see if an event has occurred. The more file descriptors to loop over, the longer it takes, just like with our restaurant example. The evented systems on the other hand, gets notified from the kernel when something interesting happens with a file descriptor, so the number of file descriptors does not really affect the overall performance. After that big drawn out explanation you are probably wondering what this has to do with the reactor pattern? Well, if you look at the diagram above again, our event-driven systems are basically providing that same model. libuv's main job is abstract out the different implementation on each OS as well as provide an event loop to schedule these events and handle other syscall calls that could gum up our nonblocking design. Instead of providing code examples like we normally do in other articles, I am going to provide a link a resource that covers programming with libuv at length (No need to document what has already been done!).

So reading this it sounds like a dream come true! Easy concurrency and web scale without the headache of threading. Unfortunately like all design patterns, there are limitations. One of the obvious ones is that the inverted program flow control of the event loop can make things difficult to debug. Another big one is that the code has to asynchronous. If we block our runloop at all, we won't be able to get updated about our other file descriptor, killing performance for our other connected clients. In practice there are just some syscalls that can't be asynchronous. Luckily, libraries like libuv actually implement a wrapper around the OS threads to workaround this issue. Even though the reactor pattern is single threaded by most implementations, it is capable of being multi-threaded, as seen with Node.js experimental cluster module.

With that, I will end our little adventure through the reactor pattern. I barely scratch the surface on what some libraries are doing with the reactor pattern. I hope through you were able to see what a powerful design pattern this can be. It is no wonder that every popular scripting language has this pattern implemented in a library. It provides a great way to handle web request (which most scripting are being used for) and because of it's single threaded nature, allows it provide high concurrency rates without threads (which most scripting languages default interpreters prevent). I know we were little short on code with this article (read zero) which probably made things a little dry. Ultimately with how popular this pattern is getting, I would encourage you to explore the resources in more depth as I'm sure it will be time well spent. As always, any questions feel free to give me a shout out on [Twitter](https://twitter.com/acmacalister).

[Reactor Pattern](http://www.cs.wustl.edu/~schmidt/PDF/reactor-siemens.pdf)

[C10K](http://www.kegel.com/c10k.html)

[Intro to libuv](http://nikhilm.github.io/uvbook/index.html)

[libuv](https://github.com/joyent/libuv)

[eventmachine](https://github.com/eventmachine/eventmachine)

[Twisted](https://twistedmatrix.com/trac/)

[Pattern Languages of Program Design](http://www.amazon.com/Pattern-Languages-Program-Design-Coplien/dp/0201607344)

[epoll vs select](http://amsekharkernel.blogspot.com/2013/05/what-is-epoll-epoll-vs-select-call-and.html)

[kqueue, epoll, select](http://www.eecs.berkeley.edu/~sangjin/2012/12/21/epoll-vs-kqueue.html)

[Cluster Node.js Module](http://nodejs.org/api/cluster.html)