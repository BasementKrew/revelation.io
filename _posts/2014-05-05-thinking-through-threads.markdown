---
layout: post
title:  "Thinking Through Threads"
date:   2014-05-05 08:00:00
author: "<a href='http://daltoniam.com'>Dalton Cherry</a>"
summary: "Threads. The well known backbone of concurrent code, but how much do we really know about them? This week we explore threads and their common design models, all leading up to our grand threading library, libdispatch."
tags: Objective-C, objc, thread, concurrent, concurrency, libdispatch, queue, pool
---

Last week we covered the history and high level concepts of concurrency. This week we narrow our focus on threads. The content on threads is nothing short of exhausting, so we will limit our discussion to but a few major and important topics. Lastly we will wrap up our discussion with libdispatch (also known as Grand Central Dispatch) and threading problems it solves.

True to form we start our discussion with a question, what is a thread? A thread is defined as: "A thread of execution is the smallest sequence of programmed instructions that can be managed independently by an operating system scheduler". That was pretty easy to understand. A Thread represents code that can be managed independently and schedule to run on its own. This allows our OS(Operating System) to run threads in parallel (or time-division with single cores, but who uses those anymore?!?!?). This is similar to how different programs run at the same time, but threads get the benefit of being lighter weight by being able to share resources like memory.

Now that we understand what a thread is, there are 2 major thread design models we would like to cover (of course there are more than 2, this are just the two major ones). The first model is know as the one to one model. This means one userspace thread to one kernel thread. A kernel thread is a thread implement in the kernel and scheduled on the OS scheduler. A userspace thread is a thread implemented in userspace, which means the kernel knows nothing about this thread and scheduling is handle by library's runtime. The advantage is that we have a kernel thread backing every userspace thread, so we can get true parallelism. The disadvantage is that we have a kernel thread backing every userspace thread, thus creating a lot of overhead if a large amount of threads are created.

 The other major thread design pattern is a one to many model. This means many userspace threads to one kernel thread. This of course has the opposite trade offs of the one to one model, where it might not be as parallel as the one to one model, but does not have the overhead of a kernel thread for every userspace thread created. One other bonus pattern (I lied, sorry, you will like it though!) is "green threads" or "fibers". This is what ruby uses to do its multi-threading. This pattern has no kernel threads in it, and just userspace. This of course means no true parallelism can exist, but it does allow for a simple way to schedule work between multiple tasks.

 Alright that wasn't so bad. We got through thread design patterns without much difficulty. The important to thing to understand why these different models exist. Threads have an inherit need to do what is called "context switch". This happens because we can't do all the work of every process at the same time, so the scheduler must swap out the threads so all your computer's processes can use the CPU and the other resources. Now if you have ever tried multi tasking in real, you know it can be highly inefficient for accomplishing any one task. This is what leads to our different design patterns to attempt to minimize this inherit performance impact. There are other code techniques and design patterns we can implement to assist with this problem, such as a thread pool.


 A thread pool is a way to minimize the impact of context and thread allocation overhead, by pre-creating a certain number of threads. All work is add to the pool and then shell out to each thread. If more work is added to the pool, it is queued until a thread becomes free when a task finishes. This can be visualize as when ordering food at your favorite burger joint. If the burger joint is really busy, you stand in a line (which is really a queue) and each person at the cashier takes your order. Each person in line is a task and each cashier is a thread processing a task. They work through the queue until all the people (tasks) are cleared and wait until more come around. The real question is of course, how many threads do I create? What if a task is a higher priority? What if there are programs with their own threads and thread pools using resources? These questions where formally difficult to answer until Apple introduced libdispatch or better known as Grand Central Dispatch.

  ```c++
//Thread pool example

//create our threadpool
static GPThreadPool* threadPool = new GPThreadPool(5); 
//schedule some work
THREADPARAMS* params = new THREADPARAMS;
params->somevar = 50;
threadPool->addOperation(filterThread, params);

//our background thread method
void filterThread(void* arg)
{
    THREADPARAMS* params = (THREADPARAMS*)arg;
    doWork(params->somevar);
    delete params;
}
 ```

 libdispatch or GCD(Grand Central Dispatch) created a system wide implementation of the thread pool model (as well as doing many, many other things). This allows all the application developers to simple schedule their work and libdispatch handles the rest. This gives much higher performance as libdispatch can dynamically resize the pool based on current work and use the OS's resources more efficiently, as it knows of all the tasks running, instead of just being scoped to the current program. This greatly reduces the work required to create a responsive and concurrent program, while handling the issue of context switch as efficiently as possible. Here is a quick example of GCD.

 ```objc
 //run work on a background thread
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
                id val = task.work(param);
                //now schedule a task to run on the main
                dispatch_async(dispatch_get_main_queue(),^{
                    [self finishedTask:task result:val];
                });
            });
 ```

 From this simple example, we can see how easy this really is. We where able to do some work on a background thread then do work on the main thread, all in a simple and clean API (partly thanks to blocks!). We only scratch the surface of what libdispatch can do (seamphores, signals, async I/O to name a few), I encourage you to read through the source and the Apple articles about GCD (and they aren't super boring documentation or whatever, so read them!). As always, questions, comments, feedback, and appreciated.

 Twitter: [@daltoniam](https://twitter.com/daltoniam).

- [Apple Concurrency Guide](https://developer.apple.com/library/mac/documentation/General/Conceptual/ConcurrencyProgrammingGuide/Introduction/Introduction.html)

- [libdispatch](http://libdispatch.macosforge.org/)


