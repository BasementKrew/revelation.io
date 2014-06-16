---
layout: post
title:  "Gopher Go! - Bufio"
date:   2014-06-16 08:00:00
author: "<a href='http://austincherry.me'>Austin Cherry</a>"
summary: "Continuing our golang series, we follow with the next package after archive, bufio."
tags: Go, golang, packages, pkg, bufio, buffered, unbuffered, io
---

This week in our golang series we are covering the bufio package. Before we jump straight into the package, let's have a quick overview of what buffered vs unbuffered I/O is. The key difference is the use of a buffer or not when reading or writing to a file (hence the buffered part). What we mean by with the term "buffer", is there is basically a place we read or write our bytes to before we actually make a system call to put them on a physically medium, like the hard disk. This is useful when working with large files as we buffer up our data in chunks to reduce the amount of memory usage as well as the write and read calls. With that out of the way, let's see how Go does it, by splunking through it's source.

For this article I choose to follow the `Write` call in the bufio package. Let's start with the function's prototype.

```go
func (b *Writer) Write(p []byte) (nn int, err error)
```

The method has the receiver type of Writer, takes a byte buffer and returns the number of bytes written or an error if there is one. That is pretty simple. Since it is a method on the Writer type, let's take a look at that real quick.

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

the function above is also fairly simple. Basically we do our first check to see if the bytes we are writing to the buffer is greater than the buffer has free. If so, we loop over and write those bytes out. As the comment notes above, there is a check to see if our buffer is empty. This is good as the check above already told us the bytes coming in are larger than our buffer can hold, so if the buffer is empty we can skip copying them into our buffer and just write them directly. Otherwise we copy them into our buffer and flush it out. If our bytes are smaller than our available buffer, then we skip the loop altogether and just write them to our buffer. Easy enough. Notice though, that our Writer type doesn't actually implement the the io package Writer interface, it just simply wraps it in it's own type and calls it in it's write function. This is actually a brillant idea, as let's say something with a concrete implementation of the io package Writer interface, like the File type in the os package perhaps, the ability to do buffered io without any changes. Onward to an example!

```go
package main

import (
  "bufio"
  "io"
  "log"
  "os"
)

// Process our args
func main() {
  if len(os.Args) < 3 {
    printUsage()
  } else {
    bufferFiles(os.Args[1], os.Args[2])
  }
}

// Print usage menu
func printUsage() {
  log.Println("Usage:")
  log.Println("bufio inputFile outputFile")
}

// borrowed from SO article.
// http://stackoverflow.com/questions/1821811/how-to-read-write-from-to-file
// Why fix what ain't broke?
func bufferFiles(input, output string) {
  // open input file
  fi, err := os.Open(input)
  if err != nil {
    log.Fatal(err)
  }

  // close fi on exit and check for its returned error
  defer func() {
    if err := fi.Close(); err != nil {
      log.Fatal(err)
    }
  }()

  // make a read buffer
  r := bufio.NewReader(fi)

  // open output file
  fo, err := os.Create(output)
  if err != nil {
    log.Fatal(err)
  }

  // close fo on exit and check for its returned error
  defer func() {
    if err := fo.Close(); err != nil {
      log.Fatal(err)
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
      log.Fatal(err)
    }
    if n == 0 {
      break
    }

    // write a chunk
    if _, err := w.Write(buf[:n]); err != nil {
      log.Fatal(err)
    }
  }

  if err = w.Flush(); err != nil {
    log.Fatal(err)
  }
}
```

As we can see, not much going on here. It takes an input file to read from as the first argument and then writes it out to the other filename we passed in as the second argument, chunk by chunk. The bufio package also contains a few other convenience methods for reading buffered data, but since the docs are excellent and have good example programs for that, I will leave you to explore those on your own. Buffering for the most part is a pretty simple and has been discussed since before I was born, so I won't belabor it anymore. Since this was such a simple package and I got sidetracked looking at the os and syscall packages, I have a great treat. Another article this week that covers both the os and syscall packages (check it out [here](os-syscall.html))! As always, questions, comments, hobbies and interests are welcomed.

[Twitter](https://twitter.com/AC_Macalister)

[bufio](http://golang.org/pkg/bufio/)