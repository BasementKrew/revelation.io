---
layout: post
title:  "Gopher Go! - Encoding"
date:   2014-11-13 08:00:00
author: "<a href='http://austincherry.me'>Austin Cherry</a>"
summary: "This week we will take a peak at classic encoded forms in Go."
tags: Go, golang, packages, pkg, encoding, json, xml, binary, gob
---

Encoding is a fundamental part of computing. It effects pretty much every piece of software ever written, take this XKCD for example:

![](http://imgs.xkcd.com/comics/encoding.png)

As silly as the comic might be, it reminds us of just how important it is to have a great standard library for handling different encoding that have arise over the years. Luckily, Go has more encoding packages then Batman has gadgets in his utility belt. OK, OK, maybe not quite that many, but a solid set. Most of them happen to be very easy to use as well. Take this example code:

```go
package main

import (
  "bytes"
  "encoding/gob"
  "encoding/hex"
  "encoding/json"
  "encoding/xml"
  "fmt"
)

type everything struct {
  is      string
  Awesome string
}

func main() {
  e := everything{is: "Everything is", Awesome: "Awesome!!!"}
  b, err := json.Marshal(e)
  if err != nil {
    fmt.Println("failed to encode to JSON.")
  }
  fmt.Println(string(b))

  b, err = xml.Marshal(e)
  if err != nil {
    fmt.Println("failed to encode to XML.")
  }
  fmt.Println(string(b))

  buffer := new(bytes.Buffer)
  enc := gob.NewEncoder(buffer)
  if err := enc.Encode(e); err != nil {
    fmt.Println("failed to encode to Binary.", err)
  }
  fmt.Println(hex.EncodeToString(buffer.Bytes()))
}
```

We are able to take a `struct` and convert it to XML, JSON, binary and hexadecimal in under 40 lines of code, pretty sweet. Since this is suppose to be a blog on things we have learned in our programming adventures, I want to note a few things you may or may not have run into. Notice that the `everything` struct does NOT have to be an exported type, but the struct fields DO. If you run this code you will notice the `is` field does not show up in the output and for good reason. The encoding packages use reflection to encode and decode values to and fro, which needs the struct fields to be settable. The `is` field is unexported, so this makes sense. Just to prove I'm not talking total nonsense (maybe a little) paste this anywhere in the `main` func above:

```go
  // be sure to import reflect package.
  s := reflect.ValueOf(&e).Elem()
  typeOfT := s.Type()
  for i := 0; i < s.NumField(); i++ {
    f := s.Field(i)
    if f.CanSet() {
      fmt.Printf("%d: %s %s = %v\n", i, typeOfT.Field(i).Name, f.Type(), f.Interface())
    } else {
      fmt.Printf("%d: %s %s\n", i, typeOfT.Field(i).Name, f.Type())
    }
  }
```

 Since we need to save reflection for a different article that is all I have for today. Go is missing some more popular formats such as YAML and TOML, but the awesome golang community has stepped up and created some great packages for those encoding as well. Links to those below. As always, questions, comments and applauses are welcomed.

[YAML](https://github.com/go-yaml/yaml)

[TOML](https://github.com/BurntSushi/toml)

[Laws of Reflection](http://blog.golang.org/laws-of-reflection)

[Twitter](https://twitter.com/acmacalister)