---
layout: post
title:  "Gopher Go! - OS & Syscall"
date:   2014-06-16 08:00:00
author: "<a href='http://austincherry.me'>Austin Cherry</a>"
author_image: "http://www.gravatar.com/avatar/4278893e11f873d60fede435f1ae08aa.png?r=x&amp;s=320"
summary: "In a rare twist of events not only two articles in the same week, but two packages in the same article! In this article we are going to spend some time pulling apart the os and syscall packages to see just what makes them tick."
tags: Go, golang, packages, pkg, os, syscall, awesomeness, source
---

ah, Operating Systems. I could discuss them for hours, but since this is a golang series let's talk about it's os package. If you look at the overview of the os package in the documentation, it is very clear what the package is providing. A platform-independent interface to operating system functionality. This generally includes permissions, file reading/writing, interacting with processes, etc. The os package also has three subpackages: exec, signal and user. Each providing additional functionality for interacting with the os. The syscall package contains most of the platform-specific calls that the os package uses to get things done. Normally for the sake of portablity we tend to avoid making direct syscalls and use the higher level APIs found in the os package if at all possible. Since both the os and syscall packages have an exhaustive list of APIs and examples, doing thet same thing in this article would be wasteful. Instead, let's step through the source and find what makes these packages tick. For our example we are going to have a peak at the `Write` function in the os package and follow the darwin/OS X "path" of implementation. First things first, let's look at the implementation of that function:

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

Nothing too crazy happening here. The interesting part of this function is the `f.write` method call. We see that the method is lowercase, so know that is a private method within the os package. If we jump over to the `unix_file.go` source and search for the write method we come up with this:

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

Again without digging too much into it, we notice the next interesting call, `syscall.Write`. If we then jump over to the syscall package and have a look at the `syscall_unix.go` file we see the write method which looks like this:

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

Again we see a private method called `write`. If we check out the `syscall_darwin.go` and `syscall_darwin_amd64.go` we don't see any go code for the write function. This is where things get interesting. If we check out the bottom of the `syscall_darwin.go` we see something that looks like this:

```go
//sys write(fd int, p []byte) (n int, err error)
```

Mhhh, that is the function we need, but a comment isn't going to do us much good. If we dig through the build scripts in syscall package we find documentation in `mkall.sh`, `mksysnum_darwin.pl`, `mkerrors.sh` and `mksyscall.pl`. The documentation informs us that those scripts are capable of auto generating syscall functions. Nice piece of engineering. We also find in the build scripts that all the auto generated files are prefixed with a Z. That being said if we jump over to `zsyscall_darwin_386.go` we find our write method!

```go
func write(fd int, p []byte) (n int, err error) {
  var _p0 unsafe.Pointer
  if len(p) > 0 {
    _p0 = unsafe.Pointer(&p[0])
  } else {
    _p0 = unsafe.Pointer(&_zero)
  }
  r0, _, e1 := Syscall(SYS_WRITE, uintptr(fd), uintptr(_p0), uintptr(len(p)))
  n = int(r0)
  if e1 != 0 {
    err = e1
  }
  return
}
```

Mhh, this method doesn't appear to do much. The big part appears to be the Syscall function. So let's see if we can find that. Again if we turn to the documentation in the build scripts we find that the Syscall method is actually written in assembly. The one we are looking for is `asm_darwin_amd64.s`. If I understood the assembly correctly, it is for the golang assembler which are based off the Plan 9 assemblers. That makes sense as both Rob Pike and Ken Thompson were major designers of golang and were part of the team that produced the Plan 9 os. After that little historical note we see that this assembly is providing a syscall function, much like the C version in various unix platforms. For those not familiar with the syscall function, I provided a link to it below. Basically it is used when you need to call a kernel method (such as write) when there is no wrapper in C. To be complete, The `SYS_WRITE` constant being passed in comes from the `zsysnum_darwin_amd64.go` which was also generated off syscall header so our Syscall function knows where to call into the kernel to run the right function.

If we go any lower we will be looking at darwin kernel source, which we probably want to save for a different article :). Questions, comments, corrections and tales of heroism welcomed.

[Twitter](https://twitter.com/acmacalister)

[os](http://golang.org/pkg/os/)

[syscall](http://golang.org/pkg/syscall/)

[C syscall](http://man7.org/linux/man-pages/man2/syscall.2.html)