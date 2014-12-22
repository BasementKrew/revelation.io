---
layout: post
title:  "Coaxing Concurrency"
date:   2014-04-28 08:00:00
author: "<a href='http://austincherry.me'>Austin Cherry</a>"
author_image: "http://www.gravatar.com/avatar/4278893e11f873d60fede435f1ae08aa.png?r=x&amp;s=320"
summary: "Concurreny is a hot topic in modern programming. In today's article we will cover some historical and possibly philosophical ideas behind concurrency. We will end our discussion with seeing how some of my favorite languages deal with the idea of concurrency."
tags: concurrency, parallelism, threading, asynchronous, C, Objective-C, Python, Go, Golang
---

Concurrency seems like something us as programmers hear every other day. Now that you have stumbled across this article you might be asking... why? Concurrency is not a new idea. Algol 68 (A programming language invented in 1968, hence the 68) included some concurrent features. 1968 was 46 years ago, that is almost half a century. Men hadn't even walked on the moon yet. Computing has grown leaps and bounds since then, so why are we still talking about this idea of concurrency? The relatively recent phenomenon comes from the rise of the multicore processors toward the tail end of the 2000s. Multicore processors have become the standard as single core processors had hit what many called a "power wall". Some might remember the abandonment of the Pentium 4 line because of it's thermal problems. Intel couldn't keep the processor cool at the higher clock rates, so they did what anyone would do, add more processors per chip. This change in hardware design resparked the dicussions of concurrent programming or more accurately worded, parallel programming. There is a lot of talk on the interwebs about the difference between parallelism and concurrency. Let me offer a simple way of how I would sum it up. Parallelism is a form of concurrency. Concurrency in it's english definition, means something is existing or happening at the same time. In computer science we might translate this as having more than one active execution context or more than one "thread of control". Parallelism on the other hand, has a bit more narrow definition. It usually means more than one task can be physically active at once, meaning we need more than one processor. Distributed systems take this one step further, by having devices that are physically separated from another in the real world, but still doing the same work. This can be visualized as a single application running across multiple servers.

Out of this history comes probably the most popular form of concurrency, threads. Pretty much every popular language right now has threads and has them in it's standard library. Arguably the most implemented version of threading is posix threads or also known as pthread. Most higher level languages are built off pthread and while there are a million examples of how to use pthreads out there, for the sake of completeness I will make it a million and one.

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>

void *say(void *args);

int main(int argc, char *argv[])
{
    pthread_t thread1;
    pthread_create( &thread1, NULL, &say, (void *)"world");
    say((void *)"hello");

   return 0;
}

void *say(void *args)
{
    char *string = (char *)args;
    for(int i = 0; i < 5; i++) {
        usleep(1000 * 100); //1000 microseconds in a millisecond.
        printf("%s\n", string);
    }
    pthread_exit(NULL);
    return NULL;
}
```

Here is higher level threading example in Python (since Python is built off of C).

```python
import threading
import time

def say(string):
  for i in range(0,5):
    time.sleep(0.05)
    print string

t = threading.Thread(target=say, args = ("world",))
t.start()

say("hello")
```

 Let's note a few things here with threads. Despite the fact they might have some clumsy syntax, threads do a pretty good job at allowing us to have concurrent code. The need for threads came from OS designers in the late 1980s and early 1990s wanting a way to avoid the weight of creating a whole new process and share an address space. Even though threads are much more lightweight then forking a whole new process, they are not without their cost. If you have several applications trying to take advantage of multiple cores on the system, they are each creating threads to run and use resources. The task scheduler on every OS can only schedule as many as threads to run simultaneously as there are cores available on the CPU. This means applications have threads using resources or just waiting to be scheduled for there turn to use the resources. This process is known as context switching and is a normal part of every OS. This of course, has many different implementations on depending on the OS. I would like to touch on a few models of these threading implementations in different languages threading libraries. Conveniently enough, the two languages I demonstrated above use the major models of threading. On most OS implementations pthread uses what is known as a one to one model. This means each user space thread is mapped to a kernel thread. This is good as it allows truly parallel operations on multi-core machines, but incurs the overview of allocating a kernel thread each time a user space thread is created. Python on the other hand, uses what some might call "green threads" or a many to one model. Simply put, this means if a thread makes a blocking system call, the entire process will be blocked. This is good as all the thread management is happening in user space making it efficient, but unable to run to run in parallel on multi-core processors. Many scripting languages will use native/pthreads but have a Global Interpreter Lock (GIL), which changes the semantics of how the language can be concurrent, but the outcome is still relatively close to that of the many to one model. The Python documentation has a good description of this, which I will link to at the end of this article. Because of this limitation, many scripting languages have turned to using the reactor/runloop pattern or preforker model found in many popular libraries such as, libuv (C library for Node.js event processing), eventmachine, libev, resque, etc. I will link off to a couple of these examples at the end of the article as well. Now that we have the basics of threads out of the way, let's move onto Apple's Grand Central Dispatch.

Grand Central Dispatch is Apple's marketing name for libdispatch. Libdispatch is a C library that is based on a thread pool pattern. A thread pool pattern is where you create a number or threads or workers to preform task and queue work in a pool that those workers need to go through. These thread pools are at the system level and all applications can interact with them, so it is able to schedule "tasks" that each application needs to run and only create as many of threads as cores available. This prevents wasting resources on unneeded threads being created and context switching unnecessarily. GCD uses libdispatch along with nifty compiler additions to create some slick syntax known as "blocks". Here is an example.

```objective-c
#import <Foundation/Foundation.h>

void say(NSString *string);

int main(int argc, const char * argv[])
{

    @autoreleasepool {
        dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0ul), ^{
            say(@"world");
        });
        say(@"hello");
    }
    return 0;
}

void say(NSString *string)
{
    for(int i = 0; i < 5; i++) {
        usleep(1000 * 100); //1000 microseconds in a millisecond.
        NSLog(@"%@", string);
    }
}
```

Let's also throw some golang stuff in there, as it has some unique concurrency considerations as well.

```cpp
package main

import (
    "fmt"
    "time"
)

func say(s string) {
    for i := 0; i < 5; i++ {
        time.Sleep(100 * time.Millisecond)
        fmt.Println(s)
    }
}

func main() {
    go say("world")
    say("hello")
}
```

Notice how both of them are treating concurrent operations. They treat the operations less like a thread that needs to be handled and scheduled and more of a "task". Like all great forms of abstraction, this reduces the programmers cognitive load from thinking about how to thread things and more about running a task in parallel. This idea of viewing parallel operations as "tasks" is becoming more popular as seen with golang including them right in the language syntax and runtime. Rounding out the list of concurrency topics, we turn our gaze toward distributed systems. Admittedly I am far from an expert when it comes to distributed computing, but it's need in modern programming is becoming more and more apparent. Languages such as Erlang that support forms of easily distributed systems are being chosen, as top tech companies need services that can scale to process astronomical large amounts of data.

In all of this, concurrent programming seems like a pretty simple idea. The syntax in the languages presented is pretty simple. We just need things to run multiple operations at the time and we do all the time, this is just informally called "multitasking". The problem isn't so much the idea of concurrent programming as the idea of sharing resources. Once our concurrent programs need to use shared resources, all sorts of issues can arise, such as race conditions, dead locks, starvation, priority inversion, etc. Tons of solutions have presented to mitigate the different issues, but ultimately nothing has proven to be a sliver bullet or secret sauce to easily writing concurrent applications. If there had been this article, like so much other literature floating around on the Internet, would have never existed. This was a pretty basic introduction to concurrency and my hope is in this article to give new programmers a good grounding in concurrent programming. I hope in future articles to explore more concurrent topics such as libdispatch and golang routines in depth, where this grounding will be needed. As always, any comments, corrections, or kudos are welcomed at [Twitter](https://twitter.com/acmacalister).

[objc.io on concurrent Apple tech](http://www.objc.io/issue-2/concurrency-apis-and-pitfalls.html)

[libdispatch](http://libdispatch.macosforge.org/)

[pthread Documentation](http://pubs.opengroup.org/onlinepubs/7908799/xsh/pthread.h.html)

[Reactor Pattern](http://en.wikipedia.org/wiki/Reactor_pattern)

[Resque](https://github.com/resque/resque)

[Python Threading](https://docs.python.org/2/library/threading.html)

[Python GIL](https://docs.python.org/2/glossary.html#term-global-interpreter-lock)