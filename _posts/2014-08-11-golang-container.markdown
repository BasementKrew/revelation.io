---
layout: post
title:  "Gopher Go! - Container"
date:   2014-08-11 08:00:00
author: "<a href='http://austincherry.me'>Austin Cherry</a>"
summary: "This week in our golang series we are going to be looking at some common data structures with some help from the container package."
tags: Go, golang, packages, pkg, data structures, container
---

Ask any first year computer science what they are currently learning and good chance they will mention a popular data structure. As we all know, data structures are a pretty fundamental to programming and have been studied quite in depth. So instead of just looking at the basic data structures in the container package, I figured we might do something a little bit more interesting and do some comparisons to see they preform in practice. If you haven't had a chance to study data structures or their time complexities, I included some links at the bottom for some primer material.

First up is the linked list. If we look at the list in the container package, we notice it implements a doubly linked list. This data structure is generally known for it's more or less constant time insertion and deletion complexities, with a not so great indexing or search.

```go
package main

import (
  "container/list"
  "fmt"
  "time"
)

const (
  vulcanQuote = "Live long and prosper."
  size        = 10000
)

func main() {
  l := list.List{}
  insertLinkList(&l)
  iterateLinkList(&l)
}

func insertLinkList(l *list.List) {
  defer timeTrack(time.Now(), "insertLinkList")
  for i := 0; i < size; i++ {
    l.PushBack(vulcanQuote)
  }
}

func iterateLinkList(l *list.List) {
  defer timeTrack(time.Now(), "iterateLinkList")
  for e := l.Front(); e != nil; e = e.Next() {
    fmt.Sprintln(e.Value)
  }
}

func timeTrack(start time.Time, name string) {
  elapsed := time.Since(start)
  fmt.Printf("%s took %s\n", name, elapsed)
}
```

The code above is pretty simple, but let's break it down. First we create our list and pass it our `insertLinkList` function to load it up with famous Vulcan quote. `iterateLinkList` loops over all the values, doing pretty much nothing with them. Notice the `defer timeTrack` line. We are using defer so that at the end of our function it will print out how long it took the function to execute. Admittedly, this probably isn't very scientific, but should at least give us a simple benchmark. Run this code and your output should look something like this:

```go
insertLinkList took 2.321788ms
iterateLinkList took 1.484607ms
```

Of course your times will vary between runs and hardware, but that is pretty quick. ~2ms for inserts and ~1.5ms for iteration. Before we move on to the next data structure, if you had never seen the list in golang code before, you are in good company. It is not a very popular container in golang for a few reasons we will hit on in the next data structure, the slice.

The slice has gone by many different names in other various programming languages. Depending on your definition, it could be called a vector, dynamic array and such. Just as a quick note, there was an actual vector package, but it was pulled before Go 1.0. Slices strength lie in the fact that is a contiguous block of memory, meaning indexing is really great and be accessed in constant time.

```go
func main() {
  s := []string{}
  insertSlice(s)
  iterateSlice(s)
}

func insertSlice(slice []string) {
  defer timeTrack(time.Now(), "insertSlice")
  for i := 0; i < size; i++ {
    slice = append(slice, vulcanQuote)
  }
}

func iterateSlice(slice []string) {
  defer timeTrack(time.Now(), "iterateSlice")
  for _, value := range slice {
    fmt.Sprintln(value)
  }
}
```

Using the same approach as in the linked list example, we are able to fill and iterate the slice. Of course we would want to append the value over and over in a tight loop like above, but for the sake of the example, bear with me. Two interesting things came of writing this example code. The first one is that append function appears to be similar to the C++ vector and double the capacity of slice when it no longer has room to append a new value. The second is a bit more shocking. Running this code will give an output something like this:

```go
insertSlice took 569.487us
iterateSlice took 162ns
```

Notice the insert into the slice. Generally the time complexity for a vector for an insertion is quite a bit slower than the linked list. Of course in both examples, we are just appending values to the end of each data structure and not in random places where the list really shines. After several runs, the slice kept coming out quite a bit faster than the linked list, which prompted me to do some research. I ended up stumbling upon a discussion in the go-nuts mailing list, that was talking about just this. I linked to it and the other article in the discussion being used for reference to the bottom. This speed decrease as well as the fact that the linked list doesn't really enforce "type safety" using the empty interface is probably a big reason for it's lack of prevalence. Next up is the map.

The map just like the slice as gone lots of different names over the years, hash, table, associative array, dictionary, etc. I would rank toward the top end of complexity to implement when it comes to basic data structures, so it is nice that golang includes one for us to use. The map is key (see what I did there) in it's ability to search, insert and delete as it all happens in constant time. Where the map tends to fall short is in in indexing or iteration. Let's see an example of this.

```go
func main() {
  m := make(map[int]string)
  // notice how the map doesn't go in as a pointer type.
  // It appears to be passed by reference by default.
  insertMap(m)
  iterateMap(m)
}

func insertMap(m map[int]string) {
  defer timeTrack(time.Now(), "insertMap")
  for i := 0; i < size; i++ {
    m[i] = vulcanQuote
  }
}

func iterateMap(m map[int]string) {
  defer timeTrack(time.Now(), "iterateMap")
  for _, v := range m {
    fmt.Sprintln(v)
  }
}
```

Same as the list and slice examples, running this code produces an output like so:

```go
insertMap took 1.665167ms
iterateMap took 2.104039ms
```

The map is faster than the list in inserts, but slower than the slice and over course is the slowest when iterating. With that we will wrap this article up. The container package also contains a heap interface and ring for circular linked listed, but the documentation has good examples, so its not really worth repeating. Of course there are quite a few other data structures we could look at, but I think the ones above tend to be the most common in everyday programming. As always, any questions, comments or criticisms are welcomed.

[Coursera Algorithms](https://www.coursera.org/course/algs4partI)

[C Algorithms and Data Structures](http://www.cprogramming.com/algorithms-and-data-structures.html)

[List vs Slice Discussion](https://groups.google.com/forum/#!topic/golang-nuts/nXYuMX55b6c)

[Why you should never use a linked list](http://kjellkod.wordpress.com/2012/02/25/why-you-should-never-ever-ever-use-linked-list-in-your-code-again/)

[Big-O Cheat Sheet](http://bigocheatsheet.com/)

[Container Package](http://golang.org/pkg/container/)

[Twitter](https://twitter.com/acmacalister)