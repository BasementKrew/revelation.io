---
layout: post
title:  "Gopher Go! - Debug"
date:   2015-03-12 08:00:00
categories: 'austin'
summary: "Today's adventures will include spelunking through Go binaries to see what interesting tidbits we turn up."
tags: Go
keywords: Go golang packages debug compilers compiling gc unix linux executables
---

Instead of our usual brief overview of a golang package with some example code, we are going to something a bit more interesting. We are going to do some amateur level pulling apart of a compiled golang program. If you have experience with compilers and computer architectures, this article might be very fundamental to you. If you haven't, buckle up and let's take a ride to the unknown!

```go
package main

import "log"

func main() {
  log.Println("Hello World!")
  l := log.New(os.Stderr, "", LstdFlags)
  l.Println("Hello World from l")
}
```

above is a super simple program we are all familiar with. If we compile and run it, it will simply print the text `Hello World!` and `Hello World from l` with the current date to standard out. As simple as this program is, it raises a perplexing question. Go doesn't really have any "namespacing" to speak of, outside of what is scoped per package. How then, can the log package have two `Println` functions? If we take a look inside the log package source code, we see that one of the `Println` functions is part of the package and the other is "scoped" under the `Logger` struct type. That is pretty straight forward, but let's dig a little deeper. In languages that provide classes, generally, classes can have methods with the same name without any conflicts. Have you ever wondered how that is possible? Let's take a look at our "friendly" (cough..) neighborhood C++ to see if he can shed some light for us.

```c++
#include <stdio.h>

void printer(const char *text);

class Logger {
  public:
    void printer(const char *text) {
      printf("%s\n", text);
    }
};

int main(int argc, char *argv[]) {
  printer("Hello World!");
  Logger l = Logger();
  l.printer("Hello World from l");
  return 0;
}

void printer(const char *text) {
  printf("%s\n", text);
}
```

Probably not the prettiest C++ in the world, but it will do the job. If we compile and run this, it will output the same things as our Go program. Now let's take a look at the mangle of this program, using a tool called `nm`.

```
nm printer
```

printer is the binary name of our C++ program. A quick note, `nm` is only available on nix systems (sorry Windows folks), but there is some good news. The Go tool has `nm` as well, so technically speaking if Go supports a platform, I would suspect that `nm` would work (I didn't get to test it on Windows, so I can't be sure). Upon running nm, you should see something like this:

```
0000000100000ee0 T __Z7printerPKc
0000000100000f10 t __ZN6Logger7printerEPKc
0000000100000000 T __mh_execute_header
0000000100000e70 T _main
                 U _memset
                 U _printf
                 U dyld_stub_binder
```

My C++ app was compiled with clang, so your output could vary depending on your compiler. The main take away is the first two lines. Notice how there is a `__Z7printerPKc` which maps to our free `printer` function and second is `__ZN6Logger7printerEPKc` which maps to our printer function inside the Logger class. Pretty neat, but what if we run `nm` against our Go program from above?

```
000000000010576c s $f32.00000000
0000000000105770 s $f32.40d00000
0000000000105774 s $f32.4b189680
0000000000105778 s $f32.bf800000
000000000010577c s $f32.cb189680
... and a whole bunch more! (Shorten for sanity)
```

That is a ton of entries! If we run the command through wc for each line (`nm printer | wc -l`) it comes out with 2570 entries total. Before we answer that question though, let's complete our first comparsion. If we grep for anything having to with the log function we should get results that look something like so:

```
nm printer | grep log

0000000000105dd0 s go.importpath.log.
0000000000039fe0 t log.(*Logger).Fatal
000000000003a070 t log.(*Logger).Fatalf
000000000003a110 t log.(*Logger).Fatalln
000000000003a400 t log.(*Logger).Flags
00000000000398a0 t log.(*Logger).Output
000000000003a1a0 t log.(*Logger).Panic
000000000003a260 t log.(*Logger).Panicf
000000000003a340 t log.(*Logger).Panicln
000000000003a530 t log.(*Logger).Prefix
0000000000039ee0 t log.(*Logger).Print
0000000000039e50 t log.(*Logger).Printf
0000000000039f60 t log.(*Logger).Println
000000000003a4a0 t log.(*Logger).SetFlags
000000000003a5e0 t log.(*Logger).SetPrefix
00000000000388a0 t log.(*Logger).formatHeader
0000000000038440 t log.New
000000000003a6a0 t log.Println
000000000003a720 t log.init
00000000001591c3 s log.initdone.
0000000000038510 t log.itoa
0000000000151c80 b log.std
0000000000005890 t runtime.(*cpuProfile).flushlog
```

Ah, now we see. Go does have some special annotation it adds to functions attached to types and therefore giving us the freedom to have functions with the same names in a single package. With that mystery solved, let us jump back to why Go has so many symbols in it's binary compared to the C++ program? Well, as you have probably heard, this has to do with Go binaries being completely statically linked. Meaning there are no shared libraries to speak. To prove that, let's us our friend `otool` (ldd on Linux).

```
otool -L printer
```

Notice it prints nothing, other than the binary's name. Let's do the same on our C++ program.

```
 /usr/lib/libc++.1.dylib (compatibility version 1.0.0, current version 120.0.0)
/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1213.0.0)
```

Would you look at, shared libraries. In OS X, libc++ is the C++ standard library and libSystem.B is all the available system calls (POSIX, OS X specific tech, etc.). Those libraries also have shared libraries they depend on, but if you run on `nm` on either of them, you can see the different functions they implement in their symbol table. As with most things in Computer Science, there are trade offs to static and shared libraries, but it is interesting to see the differences. The Go standard library gives you access to read binaries using the `debug` package, but I had some trouble getting things to work properly for creating a simple `nm` like tool for an example. Turns out the go tool version of `nm`, doesn't use the debug package either and as it is a very infrequent need to read out binary tid-bits, I doubt it is worth the effort to do much looking in to. I always find lower level parts of how languages work an interesting topic to explore. Any excuse to get to play with them is time well spent. As always, questions and comments are welcomed.

[Source of log package](http://golang.org/src/log/log.go)

[Differences between static and dynamic libraries](http://www.ilkda.com/compile/Static_Versus_Dynamic.htm)