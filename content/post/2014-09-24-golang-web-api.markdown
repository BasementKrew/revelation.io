---
title:  "Gopher Go! - Web API"
date:   2014-09-24T10:00:00-07:00
author: "<a href='http://austincherry.me'>Austin Cherry</a>"
author_image: "http://www.gravatar.com/avatar/4278893e11f873d60fede435f1ae08aa.png?r=x&amp;s=320"
summary: "In this week's article we will be building an API server for our guitar store using some popular open source packages."
tags: Go, golang, packages, pkg, net/http, http, gorm, negroni, mux, mysql, binding, render
---

Instead of our regularly scheduled standard library package, (my plan was to do the database package, but there are already lots of great documentation and examples of how to use that online) we are going to change it up this week by building a simple API server for selling guitars. Let's dive in!

```go
package main

import (
  "fmt"
  "github.com/acmacalister/skittles"
  "github.com/codegangsta/negroni"
  _ "github.com/go-sql-driver/mysql"
  "github.com/gorilla/mux"
  "github.com/jinzhu/gorm"
  "github.com/mholt/binding"
  "gopkg.in/unrolled/render.v1"
  "io"
  "log"
  "net/http"
  "os"
  "strconv"
  "time"
)

type DBHandler struct {
  db *gorm.DB
  r  *render.Render
}

type Guitar struct {
  Id        int64     `json:"id"`
  Name      string    `json:"name"`
  Brand     string    `json:"brand"`
  Year      string    `json:"year"`
  Price     int64     `json:"price"`
  Color     string    `json:"color"`
  ImageUrl  string    `json:"image_url"`
  CreatedAt time.Time `json:"created_at"`
  UpdatedAt time.Time `json:"updated_at"`
}

// Our form values we need for updating/creating a guitar.
type GuitarForm struct {
  Name  string
  Brand string
  Year  string
  Price int64
  Color string
}

// to do some validation on our input fields. File is done separately.
func (gf *GuitarForm) FieldMap() binding.FieldMap {
  return binding.FieldMap{
    &gf.Name: binding.Field{
      Form:     "name",
      Required: true,
    },
    &gf.Brand: binding.Field{
      Form:     "brand",
      Required: true,
    },
    &gf.Year: binding.Field{
      Form:     "year",
      Required: true,
    },
    &gf.Price: binding.Field{
      Form:     "price",
      Required: true,
    },
    &gf.Color: binding.Field{
      Form:     "color",
      Required: true,
    },
  }
}

const (
  defaultPerPage = 30
)

func main() {
  // setup db. We would normally load this out of a config file,
  // but for this example, it is hardset. See gist at end of article for config example.
  db, err := gorm.Open("mysql", "root@/guitarstore?parseTime=true")

  if err != nil {
    log.Fatal(skittles.BoldRed(err))
  }
  db.LogMode(true) // This would be off in production.
  defer db.Close()
  db.AutoMigrate(&Guitar{}) // nice for development, but I would probably just write a SQL script to do this.
  db.Model(&Guitar{}).AddIndex("idx_guitar_id", "id")

  r := render.New(render.Options{})
  h := DBHandler{db: &db, r: r}

  // setup a basic CRUD/REST API for our guitar store.
  router := mux.NewRouter()
  router.HandleFunc("/guitars", h.guitarsIndexHandler).Methods("GET")
  router.HandleFunc("/guitars", h.guitarsCreateHandler).Methods("POST")
  router.HandleFunc("/guitars/{id:[0-9]+}", h.guitarsShowHandler).Methods("GET")
  router.HandleFunc("/guitars/{id:[0-9]+}", h.guitarsUpdateHandler).Methods("PUT", "PATCH")
  router.HandleFunc("/guitars/{id:[0-9]+}", h.guitarsDestroyHandler).Methods("DELETE")

  n := negroni.Classic()
  n.UseHandler(router)
  n.Run(":8080")
}

// guitarsIndexHandler returns all our guitars out of the db in a paginated fashion.
func (h *DBHandler) guitarsIndexHandler(rw http.ResponseWriter, req *http.Request) {
  page := getPage(req) - 1
  perPage := getPerPage(req)
  offset := perPage * page
  var guitars []Guitar
  h.db.Limit(perPage).Offset(offset).Find(&guitars)
  if guitars == nil {
    h.r.JSON(rw, http.StatusOK, "[]") // If we have no guitars, just return an empty array, instead of null.
  } else {
    h.r.JSON(rw, http.StatusOK, &guitars)
  }
}

// guitarsShowHandler returns a single guitar from the db.
func (h *DBHandler) guitarsShowHandler(rw http.ResponseWriter, req *http.Request) {
  id := getId(req)
  guitar := Guitar{}
  h.db.First(&guitar, id)
  h.r.JSON(rw, http.StatusOK, &guitar)
}

// guitarsCreateHandler inserts a new guitar into the db.
func (h *DBHandler) guitarsCreateHandler(rw http.ResponseWriter, req *http.Request) {
  h.guitarsEdit(rw, req, 0)
}

// guitarsUpdateHandler updates a guitar in the db.
func (h *DBHandler) guitarsUpdateHandler(rw http.ResponseWriter, req *http.Request) {
  id := getId(req)
  h.guitarsEdit(rw, req, id)
}

// guitarsEdit is shared between the create and update handler, since they share most of the logic.
func (h *DBHandler) guitarsEdit(rw http.ResponseWriter, req *http.Request, id int64) {
  guitarForm := GuitarForm{}
  if err := binding.Bind(req, &guitarForm); err.Handle(rw) {
    return
  }

  // normally we would upload to S3, but for this demo, we will just write to disk. See this gist for S3 upload code.
  upload, header, err := req.FormFile("file")
  if err != nil {
    h.r.JSON(rw, http.StatusBadRequest, map[string]string{"error": "bad file upload."})
    return
  }
  file, err := os.Create(fmt.Sprintf("public/%s", header.Filename)) // we would normally need to generate unique filenames.
  if err != nil {
    h.r.JSON(rw, http.StatusInternalServerError, map[string]string{"error": "system error occured"})
    return
  }
  io.Copy(file, upload) // write the uploaded file to disk.
  imageUrl := fmt.Sprintf("http://localhost:8080/%s", header.Filename)

  guitar := Guitar{Id: id, Name: guitarForm.Name, Brand: guitarForm.Brand, Year: guitarForm.Year,
    Price: guitarForm.Price, Color: guitarForm.Color, ImageUrl: imageUrl, CreatedAt: time.Now(), UpdatedAt: time.Now()}
  h.db.Save(&guitar)
  h.r.JSON(rw, http.StatusOK, &guitar)
}

// guitarsDestroyHandler deletes a guitar from the db.
func (h *DBHandler) guitarsDestroyHandler(rw http.ResponseWriter, req *http.Request) {
  id := getId(req)
  guitar := Guitar{}
  h.db.Delete(&guitar, id)
  h.r.JSON(rw, http.StatusOK, &guitar)
}

// getId parses our id out of the url.
func getId(req *http.Request) int64 {
  vars := mux.Vars(req)
  idString := vars["id"]
  id, err := strconv.ParseInt(idString, 10, 0)
  if err != nil {
    log.Println(skittles.BoldRed(err))
  }
  return id
}

// getPage returns the page param from the url query.
func getPage(req *http.Request) int {
  return parseQueryValues(req, "page")
}

// getPerPage returns the per_page param from the url query.
func getPerPage(req *http.Request) int {
  perPage := parseQueryValues(req, "per_page")
  if perPage == 0 {
    return defaultPerPage
  }
  return perPage
}

// parseQueryValues shared parser for page & per_page url query.
func parseQueryValues(req *http.Request, value string) int {
  vals := req.URL.Query()
  val := vals[value]
  if val != nil {
    v, err := strconv.ParseInt(val[0], 10, 0)
    if err != nil {
      log.Println(skittles.BoldRed(err))
      return 0
    }
    return int(v)
  }
  return 0
}
```

The code is pretty straight forward (I might be bias since I wrote it. :)), but let's break it down from top to bottom. First thing we see after the imports are our types. Our first type is the `DBHandler` and used to encapsulate our db connection and render. The next two types are just used for modeling our guitar table in the db and form that gets posted to our API. In our `main` function we setup the db with [gorm](https://github.com/jinzhu/gorm). Gorm is a ORM library and we will see it get some more action in our endpoints. Notice we also setup our [render](https://github.com/unrolled/render) package. Render is a simple library for rendering templates, JSON, XML, etc. We use in our endpoints to render our `Guitar` structs as json. Now that our `DBHandler` is all setup and read to go, we can use [mux](https://github.com/gorilla/mux) to setup a basic router for RESTful endpoints. We then pass that into [negroni](https://github.com/codegangsta/negroni) which we use to serve up some basic middleware in our app. 

Now that we got an app setup and ready to serve some API, all we need is our endpoint handlers. Since the handlers are pretty standard I won't belabor them to much. Looking at `guitarsIndexHandler` the interesting piece is our simple pagination we setup. We use `getPage` and `getPerPage` functions to pull the page and perPage params off the request url and then use a simple `LIMIT` and `OFFSET` query in SQL to get our results. With the `guitarsShowHandler` the only real interesting is the `getId` function. The mux router is nice enough to store our resource params that we are able to pull out and lookup our resource by, just like in other popular web frameworks. Notice the `guitarsCreateHandler` and `guitarsUpdateHandler` share mostly the same logic wrapped up in the `guitarsEdit` function. This is because updating and creating a guitar are the same form values that we can parse out in the same manner. 

Combine this with last week's crypto article code for authenticating users and you are well on your way to having a nice RESTful API. My hope is in the next few articles to show how to add authentication, authorization, etc to this bit of code and show how you can use Swift to consume these APIs. Who knows, we might dip our toe in the Javascript pool and write a angular app to use these APIs as well :). As always any feedback is appreciated.

[Twitter](https://twitter.com/acmacalister)