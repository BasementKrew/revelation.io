---
layout: post
title:  "Gopher Go! - Crypto"
date:   2014-09-01 08:00:00
categories: 'austin'
summary: "Cryptography has made significant advances with the rise of modern computing. In today's golang article we are going to take a peek at a couple of Go crypto packages."
tags: Go
keywords: Go golang packages pkg crypto
---

Cryptography and cryptanalysis are some really cool subjects to study. I'm far from an expert, but love doing research when I have some downtime. Like many modern languages Go has no shortage of great crypto libraries. Unlike may other libraries out there, Go's crypto libraries are not built off OpenSSL, which can be a great win when issues like the infamous [heartbleed](http://heartbleed.com/) bug that can crop up from time to time. The standard library crypto package contains most of the low level crypto goodies you would expect like SHA1, MD5, AES, DES, cipher blocks, etc. The package has some decent docs on basic usage of these packages, so I won't cover them here. Instead I would like to take a look at the extended crypto packages in the Go source tree, namely `bcrypt`. If you are a Ruby on Rails programmer or OpenBSD user, chances are you have heard of bcrypt. For the RoR programmers, bcrypt is widely used for user authentication by using `has_secure_password`. For non RoR programmers `has_secure_password` is a simple wrapper around ruby bcrypt gem and can be an excellent way to encrypt user's password for a web application. With that brief description out of the way, let's take a look at implementing a version of `has_secure_password` in Go. If you haven't done so already, go ahead and pull the extra crypto packages for golang.

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
  "time"
)

// Contains our sql connection.
type DBHandler struct {
  db *sql.DB
}

func main() {
  // Open SQL connection to db.
  db, err := sql.Open("mysql", "root@/tester")
  if err != nil {
    log.Fatal(err)
  }
  defer db.Close()

  // Pass it to the handler.
  h := DBHandler{db: db}

  // A couple HTTP routes.
  http.HandleFunc("/create", h.CreateHandler)
  http.HandleFunc("/auth", h.AuthHandler)
  http.ListenAndServe(":8081", nil)
}

func (h *DBHandler) CreateHandler(rw http.ResponseWriter, req *http.Request) {
  // Get the form values out of the POST request.
  name := req.FormValue("name")
  password := req.FormValue("password")

  // Generate a hashed password from bcrypt.
  hashedPass, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.MinCost)
  if err != nil {
    log.Fatal(err)
  }

  // Stick that in our users table of our db.
  _, err = h.db.Query("INSERT INTO users (name, password_digest, created_at, updated_at) VALUES(?,?,?,?)",
    name, hashedPass, time.Now(), time.Now())
  if err != nil {
    log.Fatal(err)
  }

  // Write a silly message back to the client.
  rw.Write([]byte("Created user!"))
}

func (h *DBHandler) AuthHandler(rw http.ResponseWriter, req *http.Request) {
  // Get the form values out of the POST request.
  name := req.FormValue("name")
  password := req.FormValue("password")

  // Find the user by his name and get the password_digest we generated in the create method out.
  var digest string
  if err := h.db.QueryRow("SELECT password_digest FROM users WHERE name = ?", name).Scan(&digest); err != nil {
    log.Fatal(err)
  }

  // Compare that password_digest to our password we got from the form value.
  // If the error is not equal to nil, we know the auth failed. If there is no error, it
  // was successful.
  if err := bcrypt.CompareHashAndPassword([]byte(digest), []byte(password)); err != nil {
    rw.Write([]byte("auth failure..."))
  } else {
    rw.Write([]byte("auth successful!"))
  }
}
```

Reading the comments, it is pretty clear what is going on with this code. Just for completeness, I believe a high level description is in order. We setup two routes, one for creating a new user and the other for authenticating them. If we do an HTTP POST to the create route it will add a user, with a hashed password from bcrypt. The auth route will check the user and password we posted and see if they match for that user and write a message back depending on if the password is correct. Of course this is a simple little example of how you can use Go's crypto libraries. I also did a little work with ssh package if want to check that out that project out on Github (link below). With that I will bid a farewell. As always, any questions, comments, or criticisms are welcomed.

[SSH Example](https://github.com/acmacalister/kirk)

[Crypto Package](http://golang.org/pkg/crypto/)

[go.crypto](http://godoc.org/code.google.com/p/go.crypto)

[bcrypt](http://godoc.org/code.google.com/p/go.crypto/bcrypt)

[has\_secure\_password](http://api.rubyonrails.org/classes/ActiveModel/SecurePassword/InstanceMethodsOnActivation.html)

[Twitter](https://twitter.com/acmacalister)