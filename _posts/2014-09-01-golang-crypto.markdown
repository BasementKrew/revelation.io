---
layout: post
title:  "Gopher Go! - Crypto"
date:   2014-08-11 08:00:00
author: "<a href='http://austincherry.me'>Austin Cherry</a>"
summary: "Cryptography has made significant advances with the rise of modern computing. In today's golang article we are going to take a peek at a couple of Go crypto packages."
tags: Go, golang, packages, pkg, crypto
---

Cryptography and cryptanalysis are some really cool subjects to study. I'm far from an expert, but love doing research when I have some downtime. Like many modern languages Go has no shortage of great crypto libraries. Unlike may other libraries out there, Go's crypto libraries are not built off OpenSSL, which can be a great win when issues like the infamous [heartbleed](http://heartbleed.com/) bug in OpenSSL happens.