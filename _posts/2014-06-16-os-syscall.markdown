---
layout: post
title:  "Gopher Go! - OS & Syscall"
date:   2014-06-16 08:00:00
author: "<a href='http://austincherry.me'>Austin Cherry</a>"
summary: "In a rare twist of events not only two articles in the same week, but two packages in the same article! In this article we are going to spend some time pulling apart the os and syscall packages to see just what makes them tick."
tags: Go, golang, packages, pkg, os, syscall, awesomeness, source
---

Since we are talking about File's Write function, let's dig through that a bit to add a more complete understanding.

```go
func (f *File) Write(b []byte) (n int, err error) {
  if f == nil {
    return 0, ErrInvalid
  }
  n, e := f.write(b)
  if n < 0 {
    n = 0
  }

  epipecheck(f, e)

  if e != nil {
    err = &PathError{"write", f.name, e}
  }
  return n, err
}
```

Without getting into into the File's Write function (this is about the bufio package after all) we notice what looks like the interesting part the `f.write` method call. We see that the method is lowercase, so know that is a private method within the file package. If we jump over to the unix_file.go source and search for the write method we come up with this:

```go
func (f *File) write(b []byte) (n int, err error) {
  for {
    bcap := b
    if needsMaxRW && len(bcap) > maxRW {
      bcap = bcap[:maxRW]
    }
    m, err := syscall.Write(f.fd, bcap)
    n += m

    // If the syscall wrote some data but not all (short write)
    // or it returned EINTR, then assume it stopped early for
    // reasons that are uninteresting to the caller, and try again.
    if 0 < m && m < len(bcap) || err == syscall.EINTR {
      b = b[m:]
      continue
    }

    if needsMaxRW && len(bcap) != len(b) && err == nil {
      b = b[m:]
      continue
    }

    return n, err
  }
}
```

Again without digging too much into we notice the next interesting call in this which is the `syscall.Write`. If we then jump over to the syscall package and have a look at the syscall_unix.go file we see the write method which looks like this:

```go
func Write(fd int, p []byte) (n int, err error) {
  if raceenabled {
    raceReleaseMerge(unsafe.Pointer(&ioSync))
  }
  n, err = write(fd, p)
  if raceenabled && n > 0 {
    raceReadRange(unsafe.Pointer(&p[0]), n)
  }
  return
}
```

Again we see a private method called `write`. This is where things get interesting. The write method doesn't appear to have any Go code associated with it. Instead I had a look at the syscall_darwin.go (since I'm a mac, hence the unix file we jumped through above) and read the comments at the top.

```go
// Darwin system calls.
// This file is compiled as ordinary Go code,
// but it is also input to mksyscall,
// which parses the //sys lines and generates system call stubs.
// Note that sometimes we use a lowercase //sys name and wrap
// it in our own nicer implementation, either here or in
// syscall_bsd.go or syscall_unix.go.
```

If we scroll to the bottom of the file we can see a bunch of //sys comments with darwin and posix system calls, which include...

```go
//sys write(fd int, p []byte) (n int, err error)
```

Now this is where things got shakey for me, since I haven't done much research into go runtime or how to compile it from source. That being said I will leave you dear reader with some of handy guess work, which has always proven quite reliable :). Most of those sys comment are very familiar to me. In days of yester-year I did some C programming on Linux and OS X systems which gave me some exposure to Posix and BSD system calls, which most of those //sys golang functions appear to map to. I would think those "stubs" from the comments above is the way the golang runtime calls into the Posix and BSD APIs on the system. The biggest thing that preplexs me with the system how golang functions are being mapped directly to those C system calls. Let's take the write example from above. If we look at the current Posix standard for this we see:

```cpp
ssize_t write(int fildes, const void *buf, size_t nbyte);
```

Notice it mostly matches with the golang function, but as we know in computer science mostly is very rarely good enough. If anyone has any insight into how this works and could expand, I would be most appreciative. Otherwise, I will do research at a later date and we sure to write about my finding.

Alright, so back on topic with the bufio package. We see a great example of buffered io could be implemented on any type that implements the io.Writer interface. Since we used the File type as an example, for completeness let's give an example of buffered io with writing a file.

```go
awesome go code here
```

With that, we end our joyride through the bufio package. Questions, comments and tales of heroism welcomed.