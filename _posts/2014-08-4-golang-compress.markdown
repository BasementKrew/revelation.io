---
layout: post
title:  "Gopher Go! - Compress"
date:   2014-07-14 08:00:00
author: "<a href='http://austincherry.me'>Austin Cherry</a>"
summary: "This week in our golang series we will be taking a look at the bytes and strings packages."
tags: Go, golang, packages, pkg, compress, compression, bzip2, flate, gzip, lzw, zlib
---

Ahh, data compression. What nerd doesn't love a freshly compressed bag of bits in the morning to kick start the day or maybe I'm getting that confused with coffee... Any who, in today's article we are going to take a look at the compression package in Go. Since compression is a pretty well worn topic and does really need an introduction, let's jump right in with how it is used in the golang standard library. The first thing you might notice when opening up the compress package documentation is there are three fairly common compression formats of bzip2, gzip and zlib, plus two not so fairly common ones, flate and lzw. For those not on the up and up of the compression game these are two common data compression algorithms used by a ton of different file types. The Wikipedia articles have a good overview of these algorithms, so I won't belabor them here. There is one interesting thing to note here. Even though you probably won't be using the flate or lzw packages directly (unless maybe you are writing a PDF reader), the other higher level gzip, bzip2 and zlib use the flate package, while the lzw package is used by the image/gif package in the standard library. Quite convenient. Since there really isn't much else to touch on when it comes to using the compress package, I figured why not write a quick and dirty command line utility that can decompress files using the three provided compression packages?

```go
package main

import (
  "bufio"
  "bytes"
  "compress/bzip2"
  "compress/gzip"
  "compress/zlib"
  "fmt"
  "io"
  "io/ioutil"
  "log"
  "os"
  "strings"
)

func main() {
  processCommandLine()
}

func processCommandLine() {
  if len(os.Args) < 3 {
    printUsage()
  } else {
    cmd := os.Args[1]
    switch cmd {
    case "gzip":
      gzipFiles()
    case "bzip":
      bzipFiles()
    case "zlibFiles":
      zlibFiles()
    default:
      printUsage()
    }
  }
}

func gzipFiles() {
  b := readFile()
  r, err := gzip.NewReader(b)
  if err != nil {
    log.Fatal(err)
  }
  defer r.Close()
  filename := strings.TrimSuffix(os.Args[2], ".gz")
  writeFile(filename, r)
}

func bzipFiles() {
  b := readFile()
  r := bzip2.NewReader(b)
  filename := strings.TrimSuffix(os.Args[2], ".bz2")
  writeFile(filename, r)
}

func zlibFiles() {
  b := readFile()
  r, err := zlib.NewReader(b)
  if err != nil {
    log.Fatal(err)
  }
  defer r.Close()
  filename := strings.TrimSuffix(os.Args[2], ".zlib")
  writeFile(filename, r)
}

func writeFile(filename string, r io.Reader) {
  fo, err := os.Create(filename)
  if err != nil {
    log.Fatal(err)
  }

  defer func() {
    if err := fo.Close(); err != nil {
      log.Fatal(err)
    }
  }()

  w := bufio.NewWriter(fo)
  w.ReadFrom(r)
  if err = w.Flush(); err != nil {
    log.Fatal(err)
  }
}

func readFile() *bytes.Reader {
  buffer, err := ioutil.ReadFile(os.Args[2])
  if err != nil {
    log.Fatal(err)
  }
  return bytes.NewReader(buffer)
}

func printUsage() {
  fmt.Println("Usage:")
  fmt.Println("compress gzip file.gz")
  fmt.Println("compress bzip file.bz2")
  fmt.Println("compress zlib file.zlib")
}
```

Just like that we are compressing and decompressing files! Throw that together with the tar and zip example from our [archive](golang-archive.html) article and you can replace a whole slew of command line utilities! As always, any thoughts or comments are appreciated.

[LZW](http://en.wikipedia.org/wiki/Lempel%E2%80%93Ziv%E2%80%93Welch)

[DEFLATE](http://en.wikipedia.org/wiki/DEFLATE)

[Compress Package](http://golang.org/pkg/compress/)

[Twitter](https://twitter.com/acmacalister)