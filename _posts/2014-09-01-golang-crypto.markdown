---
layout: post
title:  "Gopher Go! - Crypto"
date:   2014-08-11 08:00:00
author: "<a href='http://austincherry.me'>Austin Cherry</a>"
summary: "Cryptography has made significant advances with the rise of modern computing. In today's golang article we are going to take a peek at a couple of Go crypto packages."
tags: Go, golang, packages, pkg, crypto
---

Cryptography and cryptanalysis are some really cool subjects to study. I'm far from an expert, but love doing research when I have some downtime. Like many modern languages Go has no shortage of great crypto libraries. Unlike may other libraries out there, Go's crypto libraries are not built off OpenSSL, which can be a great win when issues like the infamous [heartbleed](http://heartbleed.com/) bug in OpenSSL crop up from time to time. The standard library crypto package contains most of the low level crypto goodies you would expect like SHA1, MD5, AES, DES, cipher blocks, etc. The package has some decent docs on basic usage of these packages, so I won't cover them here. Instead I would like to take a look at the extended crypto packages in the Go source tree, namely `bcrypt`. If you are a Ruby on Rails programmer or OpenBSD user, chances are you have heard of bcrypt. For the RoR programmers, bcrypt is widely used for user authentication by using `has_secure_password`. For non RoR programmers `has_secure_password` is a simple wrapper around ruby bcrypt gem and can be an excellent way to encrypt users password for a web application. With that brief description out of the way, let's take a look at implementing a version of `has_secure_password` in Go. If you haven't done so already, go ahead and pull the extra crypto packages for golang.

`go get code.google.com/p/go.crypto`

Next the code.

```go
package main

import (
  "code.google.com/p/go.crypto/bcrypt"
  "database/sql"
  _ "github.com/go-sql-driver/mysql"
  "log"
  "net/http"
)

type DBHandler struct {
  db *sql.DB
}

func main() {
  db, err := sql.Open("mysql", "root@/mobile_lsfilter_dev")
  if err != nil {
    log.Fatal(err)
  }
  defer db.Close()
  h := DBHandler{db: db}

  http.HandleFunc("/", h.HomeHandler)
  http.ListenAndServe(":8081", nil)
}

func (h *DBHandler) HomeHandler(rw http.ResponseWriter, req *http.Request) {
  name := "dalton"
  password := []byte("test")
  var digest string
  if err := h.db.QueryRow("SELECT password_digest FROM users WHERE name = ?", name).Scan(&digest); err != nil {
    log.Fatal(err)
  }

  if err := bcrypt.CompareHashAndPassword([]byte(digest), password); err != nil {
    rw.Write([]byte("auth failure..."))
  } else {
    rw.Write([]byte("auth successful!"))
  }
}
```