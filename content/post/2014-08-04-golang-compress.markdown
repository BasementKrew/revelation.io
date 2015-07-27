---
title:  "Gopher Go! - Compress"
date:   2014-08-04T10:00:00-07:00
author: "<a href='http://austincherry.me'>Austin Cherry</a>"
author_image: "http://www.gravatar.com/avatar/4278893e11f873d60fede435f1ae08aa.png?r=x&amp;s=320"
summary: "This week in our golang series we will be taking a look at the compress package."
tags: Go, golang, packages, pkg, compress, compression, bzip2, flate, gzip, lzw, zlib
---

Ahh, data compression. What nerd doesn't love a freshly compressed bag of bits in the morning to kick start the day or maybe I'm getting that confused with coffee... Any who, in today's article we are going to take a look at the compression package in Go. Since compression is a pretty well worn topic and does really need an introduction, let's jump right in with how it is used in the golang standard library. The first thing you might notice when opening up the compress package documentation is there are three fairly common compression formats of bzip2, gzip and zlib, plus two not so fairly common ones, flate and lzw. For those not on the up and up of the compression game these are two common data compression algorithms used by a ton of different file types. The Wikipedia articles have a good overview of these algorithms, so I won't belabor them here. There is one interesting thing to note here. Even though you probably won't be using the flate or lzw packages directly (unless maybe you are writing a PDF reader), the other higher level gzip, bzip2 and zlib use the flate package, while the lzw package is used by the image/gif package in the standard library. Quite convenient. Since there really isn't much else to touch on when it comes to using the compress package, I figured why not write a quick and dirty command line utility that can decompress files using the three provided compression packages?

```go
package main

import (
  "compress/bzip2"
  "compress/gzip"
  "compress/zlib"
  "fmt"
  "io"
  "log"
  "os"
  "strings"
)

func main() {
  processCommandLine()
}

//Process Command Line Args.
func processCommandLine() {
  if len(os.Args) < 3 {
    printUsage()
  } else {
    file, err := os.Open(os.Args[2])
    if err != nil {
      log.Fatal(err)
    }
    cmd := os.Args[1]
    switch cmd {
    case "gzip":
      gzipFiles(file)
    case "bzip":
      bzipFiles(file)
    case "zlibFiles":
      zlibFiles(file)
    default:
      printUsage()
    }
  }
}

// Create new Reader for gzip files.
func gzipFiles(f *os.File) {
  in, err := gzip.NewReader(f)
  if err != nil {
    log.Fatal(err)
  }
  decompress(in, ".gz")
}

// Create new Reader for bzip2 files.
func bzipFiles(f *os.File) {
  in := bzip2.NewReader(f)
  decompress(in, ".bz2")
}

// Create new Reader for zlib files.
func zlibFiles(f *os.File) {
  in, err := zlib.NewReader(f)
  if err != nil {
    log.Fatal(err)
  }
  decompress(in, ".zlib")

}

// Create a new file and write the contents of the
// compressed file to it.
func decompress(r io.Reader, ext string) {
  trimmed := strings.TrimSuffix(os.Args[2], ext)
  out, err := os.Create(trimmed)
  if err != nil {
    log.Fatal(err)
  }
  _, err = io.Copy(out, r)
  if err != nil {
    log.Fatal(err)
  }
  out.Close()
}

// Prints a simple help menu.
func printUsage() {
  fmt.Println("Usage:")
  fmt.Println("decompress gzip file.gz")
  fmt.Println("decompress bzip file.bz2")
  fmt.Println("decompress zlib file.zlib")
}

```

Just like that we are decompressing files! Throw that together with the tar and zip example from our [archive](golang-archive.html) article and you can replace a whole slew of command line utilities! As always, any thoughts or comments are appreciated.

[LZW](http://en.wikipedia.org/wiki/Lempel%E2%80%93Ziv%E2%80%93Welch)

[DEFLATE](http://en.wikipedia.org/wiki/DEFLATE)

[Compress Package](http://golang.org/pkg/compress/)

[Twitter](https://twitter.com/acmacalister)