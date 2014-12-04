---
layout: post
title:  "Gopher Go! - Template"
date:   2014-12-04 08:00:00
author: "<a href='http://austincherry.me'>Austin Cherry</a>"
summary: "Simple look at server side templating in Go."
tags: Go, golang, packages, pkg, templating, mustache, rendering
---

Templating is a pretty popular subject in web development today. If you have done any amount of web development, chances are you have worked with templates. The most popular types of templates that come to my mind are ERB templates in Ruby on Rails and Mustache templates. Below is a small sample of `html/template` and `mustache` templates in Go.

```go
package main

import (
  "github.com/hoisie/mustache"
  "html/template"
  "log"
  "net/http"
  "time"
)

type Guitar struct {
  Id        int64     `json:"id"`
  Name      string    `json:"name"`
  Brand     string    `json:"brand"`
  Year      int64     `json:"year"`
  Price     int64     `json:"price"`
  Color     string    `json:"color"`
  ImageUrl  string    `json:"image_url"`
  CreatedAt time.Time `json:"created_at"`
  UpdatedAt time.Time `json:"updated_at"`
}

func main() {
  g := Guitar{Id: 1, Name: "Les Paul", Brand: "Gibson", Year: 1966, Price: 3500, Color: "Sunburst Cherry",
    CreatedAt: time.Now(), UpdatedAt: time.Now()} // normally we would pull this out of a db. See the API article on how to.
  http.HandleFunc("/", g.templateHandler)
  http.HandleFunc("/mustache", g.mustacheHandler)
  http.ListenAndServe(":8080", nil)
}

func (g *Guitar) templateHandler(rw http.ResponseWriter, req *http.Request) {
  t, err := template.ParseFiles("guitar.html")
  if err != nil {
    log.Fatal(err)
  }
  t.Execute(rw, g) // notice guitar is a pointer.
}

func (g *Guitar) mustacheHandler(rw http.ResponseWriter, req *http.Request) {
  rw.Write([]byte(mustache.RenderFile("guitar.mustache", g))) // notice guitar is a pointer.
}
```

Pretty simple server code. Let's check out the template files. First is the `html/template` example:

```html
<html>
  <head>
    <title>{{.Brand}} {{.Name}} - {{.Color}}</title>
  </head>
  <body>
    <h1>{{.Color}} {{.Year}} {{.Brand}} {{.Name}} - ${{.Price}}</h1>
  <body>
</html>
```

Second is the mustache template:

```html
<html>
  <head>
    <title>{{Brand}} {{Name}} - {{Color}}</title>
  </head>
  <body>
    <h1>{{Color}} {{Year}} {{Brand}} {{Name}} - ${{Price}} {{Bull}}</h1>
  <body>
</html>
```

There are a couple things to note above. First we are re-rendering the template each time the route is called. This is great for development, but we would want to cache those in production. Another thing is to take care with `html/template`. If the template is incorrect or has any invalid fields, it will cause a panic. The Mustache library seems to just ignore errors in the template. This probably wouldn't be much of an issue in production as you would do some like so:

```go
var templates = template.Must(template.ParseFiles("guitar.html", "other_template.html"))
```

And make sure all the templates were parsed and ready to go before starting the app up. If you made it this far, I would suggest you take a look at the golang article on "Writing Web Applications". It is chalked full of great information on getting started to write web applications using the standard library. Also Jeremy Saenz (codegangsta) "Building Web Apps with Go" on the basic of writing web apps in Go. As always, questions and comments are welcomed.

[Writing Web Applications](https://golang.org/doc/articles/wiki/)

[Building Web Apps with Go](http://codegangsta.gitbooks.io/building-web-apps-with-go/)

[html/template](http://golang.org/pkg/html/template/)

[mustache](https://github.com/hoisie/mustache)