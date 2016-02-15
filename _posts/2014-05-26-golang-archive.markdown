---
layout: post
title:  "Gopher Go! - Archive"
date:   2014-05-26 08:00:00
author: "<a href='http://austincherry.me'>Austin Cherry</a>"
author_image: "http://www.gravatar.com/avatar/4278893e11f873d60fede435f1ae08aa.png?r=x&amp;s=320"
summary: "This week I will be kicking off what some might call a new series. Starting today, each week I write, I will be reviewing a package out of the Go standard library."
tags: Go, golang, packages, pkg, archive, tar, zip
---

It's no secret that I have been actively learning golang. I have to say it is probably one of my favorite languages to date. I have always wanted a language that's syntax and speed were close to the ballpark of C, but easy enough to write like other high productivity languages such as Ruby, Python, Java, etc. I have had many discussions with my brother on possibly writing a language like this and when I started exploring Go, I feel like it just "fits the bill". Unfortunately, being an iOS developer at my current company and our web team being a devout Ruby on Rails shop, my time to spend with Go is mostly limited to side projects. That being said, I thought there would be no better way to really sink my teeth into the language then to sit down and explore each package in the Go standard library. Since I'm just starting this adventure I figured I should share my finding with the rest of the world and hopefully get feedback that can improve me and the community as a whole. If you don't currently know Go, but would like to follow along with this little quest, I have included some resources at the end of this article to get you started. Now, without farther ado, let's dive in.

After a quick peruse through the packages on the golang website, there are 39 top level packages, most of which appear to be fairly well documented. Even though a lot of these are top level packages, they don't actually implement anything themselves, if we count them and their children, it comes out to 142 packages overall. I figured since this is the first article in the series, why not start with first package listed, **archive**. As noted above, the archive package like many of the golang packages, does not actually implement anything. Instead it contains two archiving libraries, tar and zip. Both tar and zip are very common file formats, so I wouldn't elaborate on those too much here.

In common practice tar and zip are easy way to store a bunch of files together into one file. Tar is pretty much a unix only thing and has been implemented a little bit different on the various unix platforms. Luckily for us, the golang team has abstracted these differences out into an easy to use API. The golang documentation has example programs on how to use both, but for the sake of completeness, below is a command line program for tar'ing and zipping files.

```go

package main

import (
  "archive/tar"
  "archive/zip"
  "fmt"
  "io/ioutil"
  "log"
  "os"
  "path"
)

type fileType struct {
  Name string
  Body []byte
}

func main() {
  processCommandLine()
}

func processCommandLine() {
  if len(os.Args) < 4 {
    printUsage()
  } else {
    cmd := os.Args[1]
    switch cmd {
    case "tar":
      tarFiles()
    case "zip":
      zipFiles()
    default:
      printUsage()
    }
  }
}

func printUsage() {
  fmt.Println("Usage:")
  fmt.Println("archive tar path file1 file2 file3")
  fmt.Println("archive zip path file1 file2 file3")
}

func tarFiles() {
  //Create a file to write to.
  tarFile, err := os.Create(os.Args[2])
  if err != nil {
    log.Fatal(err)
  }

  //Create a new tar archive.
  tarWriter := tar.NewWriter(tarFile)

  //Read the files from disk and loop over them to create the tar
  files := readFilesFromDisk()
  for _, file := range files {
    header := &tar.Header{
      Name: file.Name,
      Size: int64(len(file.Body)),
    }
    if err := tarWriter.WriteHeader(header); err != nil {
      log.Fatalln(err)
    }
    if _, err := tarWriter.Write([]byte(file.Body)); err != nil {
      log.Fatalln(err)
    }
  }

  // Make sure to check the error on Close.
  if err := tarWriter.Close(); err != nil {
    log.Fatalln(err)
  }
}

func zipFiles() {
  //Create a file to write to.
  zipFile, err := os.Create(os.Args[2])
  if err != nil {
    log.Fatal(err)
  }

  //Create a new zip archive.
  zipWriter := zip.NewWriter(zipFile)

  //Read the files from disk and loop over them to create the zip
  files := readFilesFromDisk()
  for _, file := range files {
    f, err := zipWriter.Create(file.Name)
    if err != nil {
      log.Fatal(err)
    }
    _, err = f.Write([]byte(file.Body))
    if err != nil {
      log.Fatal(err)
    }
  }

  //Make sure to check the error on Close.
  err = zipWriter.Close()
  if err != nil {
    log.Fatal(err)
  }
}

func readFilesFromDisk() []fileType {
  //Get the arguments after the program name and top level command.
  filePaths := os.Args[3:]

  //Make a channel and run a go routines to read the files.
  c := make(chan []byte)
  for _, file := range filePaths {
    go func(file string) {
      buffer, err := ioutil.ReadFile(file)
      if err != nil {
        log.Fatal(err)
      }
      c <- buffer
    }(file)
  }

  //Loop over our file paths again append them to our files list.
  files := make([]fileType, 0, len(filePaths))
  for _, filePath := range filePaths {
    _, fileName := path.Split(filePath)
    files = append(files, fileType{fileName, <-c})
  }

  return files
}
```

The example is quite short, clocking in at just under a 130 lines of source. Even though the comments and source are pretty easy to read through, let's break it down step by step. First thing we do is process our command line arguments. Go has a flag library that can implement more robust flag handling, but for the sake of simplicity, we wrote a little function to handle it ourselves. First we check to make sure we have the right amount of arguments and run a zip or tar based on the first argument passed in. If the syntax is incorrect or doesn't meet our tar or zip subcommands we print out some usage guidelines. From there we can see the tar and zip calls are almost identical for creating an archive. First we create a file on disk, pass that in to our writer function and then proceed to write out the contents of the files our user supplied us to the archive. This is fairly standard and pretty boring. Where things got more interesting for me is when writing this example, was in the **readFilesFromDisk** function. I figured since we are using Go and one of it's biggest "selling" features is concurrency, why not make reading the files we want to archive happen concurrently. Turns out, Go makes this a breeze. First thing we do is create a slice off the command line arguments, so we have just the files we need to read off disk. After that, we create a channel to which we are going to write each file buffer back to when the reading is complete. We then go ahead and loop over each of the files, firing them off onto their own go-routines. Those will merrily grind away as we move on to building our structure of files to write into archive. Notice the first think we do is call **make** to dynamically allocate an array of fileType structures onto the heap. Since we already know how many files we are going to have, we pass in the length to **make** to get the perfect size every time. Now that we have our array, we are free to loop over the filePaths adding the name and contents of each file. The **path.Split** call is just to make sure get the actual filename, in case the file is a relative path. Notice when we are appending the new file structure we are reading the data out of the channel. One really legit thing about Go's channels are that they block until they have data to read out. That is wonderful as it means no need to fuss with locks, mutexes or any of that other synchronization garbage.

Well, that about wraps up our article on the archive package. As with all example code, there is plenty of room for improvement to make it "production" worthy, but my hope is you can see how easy archiving files in Go can be. We recently just had to do some archiving (or un-archiving in our case) in an iOS, which turned out having to be done mostly in C and was not nearly as easy or straight forward as the Go program above. As always, questions, comments, ugly code induced rage, atta-boys and pats on the back are welcomed.

[Twitter](https://twitter.com/acmacalister)

[Golang Book](http://www.golang-book.com)

[Tour of Go](http://tour.golang.org)

[Archive Docs](http://www.golang.org/pkg/archive)

[iOS Tar](https://www.github.com/daltoniam/tarkit)