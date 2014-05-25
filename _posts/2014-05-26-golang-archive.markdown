---
layout: post
title:  "Gopher Go! - Archive"
date:   2014-05-26 08:00:00
author: "<a href='http://austincherry.me'>Austin Cherry</a>"
summary: "This week I will be kicking off what some might call a new series. Starting today, each week I write, I will be reviewing a package out of the Go standard library."
tags: Go, golang, packages, pkg
---

It's no secret that I have been trying to actively learn golang. I have to say it is probably one of my most favorite languages to date. I have always wanted a language that's syntax and speed where close to the ballpark of C, but easy enough to write to stay productive like in Ruby, Python, Java, etc. I had many discussions with my brother on possibly writing a language like this and when I started exploring Go, I feel like it just "fits the bill". Unforunately, being an iOS developer at my current company and them being a devot Ruby on Rails shop, my time to spend with Go is mostly limited to side projects. That being said, I thought there would be no better way to really sink my teeth into the language then to sit down and explore each package in the Go standard library. Since I just starting this adventure I figure I should share my finding with the rest of the world and hopefully get feedback that can improve the community as a whole. If you don't currently know Go, but would like to follow along with this little quest, I have included some resources at the end of this article to get you started. Now, without farther ado, let's dive in.

After a quick peruse through the packages on the golang website, there are 39 top level packages, most of which appear to be fairly well documented. Even though most of these top level packages, don't actually implement anything themselves, if we count them and there children, it comes out to 142 packages overall. I figured since this is the first article in the series, why not start with first package listed, archive. As noted above, the archive package does not actually implement anything. Instead, it contains two archiving libraries, tar and zip. Both tar and zip are very common file formats, so I wouldn't elobrate on those here. Since there are two packages here, let's start with the lesser know tar.

In common practice tar is an easy way to store a bunch of files together into one easy to use file. Tar is pretty much a unix only thing and has been implmented a little bit different on the various unix platforms. Luck enough for us the golang team has abstracted these differences out into an easy to use API. The golang documentation has a nice example program on how to use the tar package, but I figured why not step it up a few notches and do a common practice of gzipping and tar'ing a file together. The example below also shows how to zip and unzip a file, all in a nice little command line program.

```go



```

