---
layout: post
title:  "Making your own middleware in Go"
date:   2015-02-06 08:00:00
author: "<a href='http://austincherry.me'>Austin Cherry</a>"
author_image: "http://www.gravatar.com/avatar/4278893e11f873d60fede435f1ae08aa.png?r=x&amp;s=320"
summary: "Making middleware in Go."
tags: Go, golang, packages, net/http, http, middleware
---

Middleware is a popular word used in the computing landscape nowadays, but do we even know what it means? According to Google, middleware is "software that acts as a bridge between an operating system or database and applications, especially on a network". Many web frameworks provide the idea of middleware in their HTTP layers and Go is no exception. As with our router we built in the last golang article we did, making Go middleware is just as easy as implementing the `ServeHTTP` method in the `net/http` package.

```go
package main

import (
  "fmt"
  "net/http"
)

type middleware struct {
  mux      http.Handler // for the router/mux. (Gorilla, httprouter, etc)
  handlers []http.HandlerFunc // middleware funcs to run.
}

func main() {
  mux := http.NewServeMux() // create a new http mux (router)
  mux.HandleFunc("/", barHandler) // add a route.
  m := New() // create some middleware
  m.Add(fooHandler, foo2Handler) // add our middleware functions.
  m.AddMux(mux) // add our router.
  http.ListenAndServe(":8080", m) // let's run this thing.
}

// Inits a new middleware chain.
func New() *middleware {
  return &middleware{handlers: make([]http.HandlerFunc, 0, 0)}
}

// Add adds a variable number of handlers using variadic arguments.
func (m *middleware) Add(h ...http.HandlerFunc) {
  m.handlers = append(m.handlers, h...)
}

// AddMux adds our mux to run.
func (m *middleware) AddMux(mux http.Handler) {
  m.mux = mux
}

// So we can satisfy the http.Handler interface.
func (m *middleware) ServeHTTP(w http.ResponseWriter, r *http.Request) {
  for _, h := range m.handlers {
    h.ServeHTTP(w, r)
  }
  m.mux.ServeHTTP(w, r)
}

func fooHandler(rw http.ResponseWriter, r *http.Request) {
  fmt.Println("running foo.")
}

func foo2Handler(rw http.ResponseWriter, r *http.Request) {
  fmt.Println("running foo 2.")
}

func barHandler(rw http.ResponseWriter, r *http.Request) {
  fmt.Println("running bar.")
  fmt.Fprint(rw, "bar!\n")
}
```

That's all there is to it! Like I said above the magic all has to do with the `ServeHTTP` which satisfies the `http.Handler` interface. Obviously there a bunch of additions we do to this to make it production worth (move it into it's own package, add ways to stop the middleware chain at a certain point, convenience methods, etc), but all in all this should give us a basic idea of how middleware is setup in Go. As always, questions and comments are welcomed.