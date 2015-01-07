---
layout: post
title:  "Build your own Router in Go"
date:   2015-1-07 08:00:00
author: "<a href='http://austincherry.me'>Austin Cherry</a>"
author_image: "http://www.gravatar.com/avatar/4278893e11f873d60fede435f1ae08aa.png?r=x&amp;s=320"
summary: "Simple look at server side templating in Go."
tags: Go, golang, packages, net/http, http, router, mux
---

As stated in the title of this article, we are going to build a router (also known as a mux) in Go. Now you might be asking why? There are a plethora of great routers out there so why "reinvent the wheel"? As a mentor of mine use to say "to learn how wheels work". So without further ado, let's jump in.

```go
/// Handle is just like "net/http" Handlers, only takes params.
type Handle func(http.ResponseWriter, *http.Request, url.Values)

// Router name says it all.
type Router struct {
  tree        *node
  rootHandler Handle
}
```

First we setup a few simple types we will use.

```go
// New creates a new router. It takes the root (fall through) route
// like how the default mux works. The only difference, you get to specify one.
func New(rootHandler Handle) *Router {
  node := node{component: "/", isNamedParam: false, methods: make(map[string]Handle)}
  return &Router{tree: &node, rootHandler: rootHandler}
}
```

Next is our `New` function which creates a router for us to add some routes to. Notice with our implementation we decided to create a "root" or "fall-through" handler, so any undefined route our router gets will have somewhere to go. This of course is not required, but just added for simplicity sake. Now that we have a router we need a function for handling actual routes.

```go
// Handle takes an http handler, method, and pattern for a route.
func (r *Router) Handle(method, path string, handler Handle) {
  if path[0] != '/' {
    panic("Path has to start with a /.")
  }
  r.tree.addNode(method, path, handler)
}
```

`Handle` is real simple. it does a quick check to make sure our "path" (URL) is valid, then adds to our tree (more on that below). The only thing left for router is implement the `ServeHTTP` from `net/http`.

```go
func (r *Router) ServeHTTP(w http.ResponseWriter, req *http.Request) {
  req.ParseForm()
  params := req.Form
  node, _ := r.tree.traverse(strings.Split(req.URL.Path, "/")[1:], params)
  if handler := node.methods[req.Method]; handler != nil {
    handler(w, req, params)
  } else {
    r.rootHandler(w, req, params)
  }
}
```

In plain language it is what allows our `Handle` function we defined to actually receive HTTP request sent to it. 

Now you might be wondering about that `tree` and `node` nonsense we have been waiting to cover. For the use case of our router, the `tree` is a Trie data structure. If you aren't familiar with a Trie, you can read more about it [here](https://www.cs.bu.edu/teaching/c/tree/trie/). There is of course, a lots of data structures you could use to build a router, but a Trie seemed like a natural fit with the structure of URLs.

```go
// node represents a struct of each node in the tree.
type node struct {
  children     []*node
  component    string
  isNamedParam bool
  methods      map[string]Handle
}
```

Just like with the router, we start by creating a type we use to represent a node in our tree. Now we need a way to add nodes to our tree.

```go
// addNode - adds a node to our tree. Will add multiple nodes if path
// can be broken up into multiple components. Those nodes will have no
// handler implemented and will fall through to the default handler.
func (n *node) addNode(method, path string, handler Handle) {
  components := strings.Split(path, "/")[1:]
  count := len(components)

  for {
    aNode, component := n.traverse(components, nil)
    if aNode.component == component && count == 1 { // update an existing node.
      aNode.methods[method] = handler
      return
    }
    newNode := node{component: component, isNamedParam: false, methods: make(map[string]Handle)}

    if len(component) > 0 && component[0] == ':' { // check if it is a named param.
      newNode.isNamedParam = true
    }
    if count == 1 { // this is the last component of the url resource, so it gets the handler.
      newNode.methods[method] = handler
    }
    aNode.children = append(aNode.children, &newNode)
    count--
    if count == 0 {
      break
    }
  }
}
```

The code is pretty simple, but might seem a little strange at first glance, so let's break it down. `addNode` starts by splitting apart the URL (call path here) into components. Once we have our components, we loop over each one, updating or adding new nodes as needed. There is an edge case here, if the component is a named parameter. The `isNamedParam` is a simple way we can support URLs like so: `/users/:name/blog` without a big fancy regex. Notice the bulk of the work is done by the `traverse` method which gets us the proper node we need to do our addition or update.

```go
// traverse moves along the tree adding named params as it comes and across them.
// Returns the node and component found.
func (n *node) traverse(components []string, params url.Values) (*node, string) {
  component := components[0]
  if len(n.children) > 0 { // no children, then bail out.
    for _, child := range n.children {
      if component == child.component || child.isNamedParam {
        if child.isNamedParam && params != nil {
          params.Add(child.component[1:], component)
        }
        next := components[1:]
        if len(next) > 0 { // http://xkcd.com/1270/
          return child.traverse(next, params) // tail recursion is it's own reward.
        } else {
          return child, component
        }
      }
    }
  }
  return n, component
}
```

As stated in the comments above, `traverse` is a recursive function that steps through our tree looking for the matching component of our URL, so it return the node we need to work with. That's it! We now have a fully functional router that we can use like so:

```go
package main

import (
  "fmt"
  "github.com/acmacalister/helm"
  "net/http"
  "net/url"
)

func main() {
  r := helm.New(Root)
  r.Handle("GET", "/", Root)
  r.Handle("GET", "/users", Users)
  r.Handle("GET", "/users/:name", UserShow)
  r.Handle("GET", "/users/:name/blog/new", UserBlogShow)
  http.ListenAndServe(":8080", r)
}

func Root(w http.ResponseWriter, r *http.Request, params url.Values) {
  fmt.Fprint(w, "Root!\n")
}

func Users(w http.ResponseWriter, r *http.Request, params url.Values) {
  fmt.Fprint(w, "Users!\n")
}

func UserShow(w http.ResponseWriter, r *http.Request, params url.Values) {
  fmt.Fprintf(w, "Hi %s", params["name"])
}

func UserBlogShow(w http.ResponseWriter, r *http.Request, params url.Values) {
  fmt.Fprintf(w, "This is %s Blog", params["name"])
}
```

I hope you enjoyed reading this as much as I did writing and researching it. Of course there quite a few changes and optimization that can be made to this, but the purpose was to give you a basic idea of how the routers in the open source community are implemented. You can find a link to the full source code below. As always, questions and comments are welcomed. Find me on Twitter at [@acmacalister](https://twitter.com/acmacalister).

[Helm](https://github.com/acmacalister/helm)

[ServeHTTP](http://golang.org/pkg/net/http/#HandlerFunc.ServeHTTP)

[Trie](https://www.cs.bu.edu/teaching/c/tree/trie/)