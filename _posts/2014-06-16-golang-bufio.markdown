---
layout: post
title:  "Gopher Go! - Bufio"
date:   2014-06-16 08:00:00
author: "<a href='http://austincherry.me'>Austin Cherry</a>"
summary: "Continuing our golang series, we follow with the next package after archive, bufio."
tags: Go, golang, packages, pkg, bufio, buffered, unbuffered, io
---

This week in our golang series we are covering the bufio package. Before we jump straight into the package, let's have a quick overview of what buffered vs unbuffered I/O is. The difference is basically the use of a buffer or not when read or writing to a file. (hence the buffered part.) What we mean by buffer is there is basically a place we read or write our bytes to before we actually make a system call to put them on a physically medium, like the hard disk. This isn't always what we want as it means that the data might not be immediately available as generally we want to fill our buffer up to reduce our read or write calls, but can be useful in many situations such as writing a large file to disk. With that out of the way, let's see how Go does it, by splunking through it's source.

For this article I chose to follow the Write call in the bufio package down it's rabbit hole, but many of the other functions in this package are very similar. Let's start with the function's prototype.

```go
func (b *Writer) Write(p []byte) (nn int, err error)
```

So it method with the receiver type Writer, it takes a byte buffer/array and returns the number of bytes write and error if there is one. Pretty simple. Since it is a method on the Writer type, let's take a look at that real quick.

```go
type Writer struct {
    err error
    buf []byte
    n   int
    wr  io.Writer
}
```

So as we above the Writer type above is implementing buffering on the io.Writer interface, which looks like this:

```go
type Writer interface {
        Write(p []byte) (n int, err error)
}
```

Which leads us to the actual Write method in the bufio package:

```go
func (b *Writer) Write(p []byte) (nn int, err error) {
  for len(p) > b.Available() && b.err == nil {
    var n int
    if b.Buffered() == 0 {
      // Large write, empty buffer.
      // Write directly from p to avoid copy.
      n, b.err = b.wr.Write(p)
    } else {
      n = copy(b.buf[b.n:], p)
      b.n += n
      b.flush()
    }
    nn += n
    p = p[n:]
  }
  if b.err != nil {
    return nn, b.err
  }
  n := copy(b.buf[b.n:], p)
  b.n += n
  nn += n
  return nn, nil
}
```

the function above is pretty simple. Basically we do our first check to see if the bytes we are writing to our buffer are greater than our buffer has free. If so we loop over and write those bytes out. As the comment notes above, there is a check to see if our buffer is empty. This is good as the check above, already told us the bytes coming in are larger than our buffer can hold, so we can skip copying them into our buffer and just write them directly. Otherwise we copy them into our buffer and flush it out. If our bytes are smaller than our available buffer, then we skip the loop altogether and just write them to our buffer. Easy enough. Notice though, that our Writer type doesn't actually implement the the io package Writer interface, it just simply wraps it in it's own type and calls it in it's write function. This is actually a brillant idea, as let's say something with a concrete implementation of the io package Writer interface, like File type in the os package perhaps, wants some buffered io. That no biggie, sense it implements io Writer interface and that is what we call to in our write method, the File type will know what to do with it. On for an example!

```go
package main

import (
  "bufio"
  "io"
  "os"
)

func main() {
  // open input file
  fi, err := os.Open("input.txt")
  if err != nil {
    panic(err)
  }
  // close fi on exit and check for its returned error
  defer func() {
    if err := fi.Close(); err != nil {
      panic(err)
    }
  }()
  // make a read buffer
  r := bufio.NewReader(fi)

  // open output file
  fo, err := os.Create("output.txt")
  if err != nil {
    panic(err)
  }
  // close fo on exit and check for its returned error
  defer func() {
    if err := fo.Close(); err != nil {
      panic(err)
    }
  }()
  // make a write buffer
  w := bufio.NewWriter(fo)

  // make a buffer to keep chunks that are read
  buf := make([]byte, 1024)
  for {
    // read a chunk
    n, err := r.Read(buf)
    if err != nil && err != io.EOF {
      panic(err)
    }
    if n == 0 {
      break
    }

    // write a chunk
    if _, err := w.Write(buf[:n]); err != nil {
      panic(err)
    }
  }

  if err = w.Flush(); err != nil {
    panic(err)
  }
}
```

As we can see, not much going on here. It takes an input file to read from as the first argument and then writes it out to the other filename we passed in as the second argument, chunk by chunk. The bufio package also contains a few other convenience methods for reading buffered data, but since the docs are excellent on that and have good example programs for that, I will leave you to explore those on your own. Buffering for the most part is a pretty simple and has beeen discussed since before I was born, so I won't belabor it anymore. Since this was such a simple package and I got sidetracked looking at the os and syscall packages, I have a great treat. Another article this week that covers both the os and syscall packages! Check it out here. As always, questions, comments, hobbies and interests are welcomed.

[Twitter](https://twitter.com/AC_Macalister)

[bufio](http://golang.org/pkg/bufio/)