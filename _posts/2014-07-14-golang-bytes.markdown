---
layout: post
title:  "Gopher Go! - Bytes & Strings"
date:   2014-07-14 08:00:00
author: "<a href='http://austincherry.me'>Austin Cherry</a>"
summary: "This week in our golang series we will be taking a look at the bytes and strings packages."
tags: Go, golang, packages, pkg, strings, bytes, unicode, utf8
---

Bits, Bytes, Strings, Characters, Encoding and the like are interesting subject to explore. Depending on which communities you hang out in, answers to these questions are likely to vary. I was originally thinking of just covering the bytes package, but since the strings and bytes packages share so many APIs in common, it didn't make sense to me cover one without the other. So without farther ado, let's jump right in.

For me, one of the first things I noticed when programming in Go was how the strings are immutable. This probably only comes as a shock to Ruby programmers, as pretty much every other modern programming language I have done work in, tends to have to immutable strings (save Ruby). Of course, this comes down to your definition of mutability, but for me (and the golang authors. See [here](http://golang.org/ref/spec#String_types)) strings are immutable. A slice of bytes on the other hand, are in fact mutable. Since C was one of the first programming languages I learned to program in, this felt pretty natural to me. Basically the difference is the same as the difference between a `const char` or `char` buffer. If you take a look at the article Rob Pike wrote on the golang blog, you find that this lines up nicely as Rob says: "a string is in effect a read-only slice of bytes". It's a good read for any aspiring golang programmer, so I will include a link to it below for you to have a look over. Just to prove we know what we are talking about, let's have a few examples:

```go
package main

import (
  "fmt"
)

func main() {
  b := []byte("dog")
  fmt.Println(string(b))
  b[0] = 'h'
  fmt.Println(string(b))
}
```

If we run this code our output is:

```
dog
hog
```

If we change this over to a string our code will look something like this:

```go
package main

import (
  "fmt"
)

func main() {
  s := "dog"
  fmt.Println(s)
  s[0] = 'h'
  fmt.Println(s)
}
```

and if we try to run it will receive an error that look something like this:

```
# command-line-arguments
./test.go:10: cannot assign to s[0]
```

Other than mutability and a few other slight differences Rob list in his article, strings and bytes are the same. Really the difference comes down to use case. Hence why the API sets of strings and bytes are so similar. Take the `Split` function for example. Here it is in the bytes package:

```go
package main

import (
  "bytes"
  "fmt"
)

func main() {
  b := []byte("a,b,c")
  sep := []byte(",")
  fmt.Printf("%q\n", bytes.Split(b, sep))
}
```

And here with the string package.

```go
package main

import (
  "fmt"
  "strings"
)

func main() {
  fmt.Printf("%q\n", strings.Split("a,b,c", ","))
}
```

Both produce the same output. For me the difference really comes down to the programmer perspective. In C, using char buffer to store a bag of bytes is a fairly common practice, but not always the most explicit on whether the characters are suppose to be human readable or not. For me, this is the biggest difference, the way we intend to use the bytes. Most of the golang standard I/O libraries return byte slices as opposed to strings, which makes sense as it could be either text or binary data depending on the file type. As with the other statements above, let's see an example:

package main

```go
package main

import (
  "fmt"
)

func main() {
  s := []byte("Cheers \xF0\x9F\x8D\xBB")
  fmt.Println(s)
}
```

This will produce a slice output of

```
[67 104 101 101 114 115 32 240 159 141 187]
```

Of course we could type conversion on this, as with a couple of the examples above when we wrap the byte buffer in a `string()` call. As opposed to string example:

```go
package main

import (
  "fmt"
)

func main() {
  s := "Cheers! \xF0\x9F\x8D\xBB"
  fmt.Println(s)
}
```

```
Cheers! 🍻
```

If your browser supports UTF8 you will see "Cheers" text with the popular clinking mugs emoji. While this is a silly and simple example, it drives home the point. strings are good for human readable text and bytes for pretty much anything else. As one last note, I would like to throw a quick recommendation for reading Joel Spolsky's blog post on Unicode and characters sets, which Rob Pike links to in the golang article below. It has a nice 10,000 foot view of Unicode and talks a bit about UTF8, which golang strings are encoded in by default. As always feedback is welcomed.

[Strings, bytes, runes and characters in Go](http://blog.golang.org/strings)

[The Absolute Minimum Every Software Developer Absolutely, Positively Must Know About Unicode and Character Sets (No Excuses!)](http://www.joelonsoftware.com/articles/Unicode.html)

[Bytes Package](http://golang.org/pkg/bytes)

[String Package](http://golang.org/pkg/strings)

[Twitter](https://twitter.com/acmacalister)