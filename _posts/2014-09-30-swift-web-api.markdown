---
layout: post
title:  "Swift API"
date:   2014-09-30 10:00:00
author: "<a href='http://daltoniam.com'>Dalton Cherry</a>"
summary: "We follow up on our article from last week by building an iOS app in Swift that interacts with our guitar store API."
tags: apple, ios, swift, http, swifthttp, json, joy, golang, web, api, guitar, store, tutorial, skeets
---

Last week [Austin](http://austincherry.me) showed how to build a standard REST API in Go. This week we build off that example and implement a Swift client to interact with the API server. Let's start off by reviewing the Go API server. I won't cover this in much detail, check out [last week's article](/golang-web-api.html) to get the goods. 

```go
package main

import (
  "code.google.com/p/go.crypto/bcrypt"
  "encoding/hex"
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
  "math/rand"
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

//Our User to auth our people
type User struct {
  Id             int64     `json:"id"`
  Name           string    `json:"name"`
  PasswordDigest string    `json:"password_digest"`
  ImageUrl       string    `json:"image_url"`
  AuthToken      string    `json:"auth_token"`
  CreatedAt      time.Time `json:"created_at"`
  UpdatedAt      time.Time `json:"updated_at"`
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

  authRouter := mux.NewRouter()
  authRouter.HandleFunc("/create", h.createUserHandler).Methods("POST")
  authRouter.HandleFunc("/login", h.loginUserHandler).Methods("POST")

  // setup a basic CRUD/REST API for our guitar store.
  router := mux.NewRouter()
  router.HandleFunc("/guitars", h.guitarsIndexHandler).Methods("GET")
  router.HandleFunc("/guitars", h.guitarsCreateHandler).Methods("POST")
  router.HandleFunc("/guitars/{id:[0-9]+}", h.guitarsShowHandler).Methods("GET")
  router.HandleFunc("/guitars/{id:[0-9]+}", h.guitarsUpdateHandler).Methods("PUT", "PATCH")
  router.HandleFunc("/guitars/{id:[0-9]+}", h.guitarsDestroyHandler).Methods("DELETE")

  //auth the guitar routes
  authRouter.Handle("/guitars", negroni.New(
    negroni.HandlerFunc(h.authHandler),
    negroni.Wrap(router),
  ))

  n := negroni.Classic()
  n.UseHandler(authRouter)
  n.Run(":8080")
}

// create a new user
func (h *DBHandler) createUserHandler(rw http.ResponseWriter, req *http.Request) {
  // Get the form values out of the POST request.
  name := req.FormValue("name")
  password := req.FormValue("password")
  imageUrl := req.FormValue("imageUrl")

  // Generate a hashed password from bcrypt.
  hashedPass, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.MinCost)
  if err != nil {
    log.Fatal(err)
  }
  count := 16
  b := make([]byte, count)
  rand.Seed(time.Now().UTC().UnixNano())
  for i := 0; i < count; i++ {
    b[i] = byte(rand.Intn(count))
  }
  token := hex.EncodeToString(b)
  // Stick that in our users table of our db.
  user := User{Name: name, PasswordDigest: string(hashedPass), ImageUrl: imageUrl, AuthToken: token, CreatedAt: time.Now(), UpdatedAt: time.Now()}
  h.db.Save(&user)
  user.PasswordDigest = "" //we don't need to expose that to the user
  h.r.JSON(rw, http.StatusOK, &user)
}

//allows an existing user to login
func (h *DBHandler) loginUserHandler(rw http.ResponseWriter, req *http.Request) {
  // Get the form values out of the POST request.
  name := req.FormValue("name")
  password := req.FormValue("password")

  user := User{}
  h.db.Where("name = ?", name).First(&user) //in production code, we would of course validate this before running a where statement on a raw value
  if err := bcrypt.CompareHashAndPassword([]byte(user.PasswordDigest), []byte(password)); err != nil {
    log.Println("login error: ", err)
    http.Error(rw, "Not authorized", http.StatusUnauthorized)
    return
  }
  user.PasswordDigest = "" //we don't need to expose that to the user
  h.r.JSON(rw, http.StatusOK, &user)
}

//middleware that checks to make sure the authToken is a valid user
func (h *DBHandler) authHandler(rw http.ResponseWriter, r *http.Request, next http.HandlerFunc) {

  token := r.FormValue("auth_token")
  user := User{}
  h.db.Where("auth_token = ?", token).First(&user) //in production code, we would of course validate this before running a where statement on a raw value
  if user.Name == "" {
    http.Error(rw, "Not authorized", http.StatusUnauthorized)
    return
  }
  next(rw, r)
}

// our guitar routes.

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

This is the basic API from last week, with a little bit of the [crypto article](/golang-crypto.html) tossed in for authentication. Again, I won't belabor, as those articles covered the gritty details. Now let's get into the client code.

First let's start with `API.swift`.

```swift
import Foundation
import SwiftHTTP //need to import so we have access to HTTPTask

//we will cover the differences of a struct and class in the next few weeks
struct API {
    ///create a new HTTPTask
    static func newTask() -> HTTPTask {
        var apiManger = HTTPTask()
        apiManger.baseURL = "http://localhost:8080"
        return apiManger
        
    }
}
```

This code is pretty straight forward. Create a new `HTTPTask` set the baseURL and return it. This leads into our next file `User.swift`

```swift
import Foundation
import SwiftHTTP
import JSONJoy

public class User: JSONJoy { //JSONJoy is a protocol
    var id: Int
    var name: String
    var imageUrl: String?
    var authToken: String
    //This is the required init method from JSONJoy. Check out the JSONJoy docs at the end of the article for more info.
    required public init(_ decoder: JSONDecoder) {
        id = decoder["id"].integer!
        name = decoder["name"].string!
        imageUrl = decoder["image_url"].string
        authToken = decoder["auth_token"].string!
    }
    
    ///do a login request
    class func login(userName: String, password: String, success:((User) -> Void)!,failure:((NSError) -> Void)!) {
        var task = API.newTask() //create a new task from the API.swift we just reviewed
        //run a POST to the "/login" route. Since we set the baseURL when creating the task this becomes "http://localhost:8080/login"
        task.POST("/login", parameters: ["name": userName, "password": password], success: { (response: HTTPResponse) in
          //the request finished. Make sure our success closure is valid
            if success != nil {
                if let resp = response.responseObject as? NSData {
                  //the object is NSData as we expected. We pass this raw JSON data to JSONDecoder which then is passed to User init method above.
                    success(User(JSONDecoder(resp)))
                }
            }
            }, { (error: NSError) in
                if failure != nil {
                    failure(error)
                }
        })
    }
    ///create an account. Basically works the same as the login
    class func create(userName: String, password: String, imageUrl: String, success:((User) -> Void)!,failure:((NSError) -> Void)!) {
        var task = API.newTask() //create a new task from the API.swift we just reviewed
        task.POST("/create", parameters: ["name": userName, "password": password,"imageUrl": imageUrl], success: { (response: HTTPResponse) in
            if success != nil {
                if let resp = response.responseObject as? NSData {
                    success(User(JSONDecoder(resp)))
                }
            }
            }, { (error: NSError) in
                if failure != nil {
                    failure(error)
                }
        })
    }
}

```

Lastly we create a new view controller and wire it up.

```swift
import UIKit

public protocol LoginDelegate {
    func didLogin(user: User) //returns the user object created in our User model (User.swift)
}

public class LoginViewController : UIViewController {
    
    @IBOutlet weak var loginButton: UIButton!
    @IBOutlet weak var createButton: UIButton!
    @IBOutlet weak var nameField: UITextField!
    @IBOutlet weak var passwordField: UITextField!
    public var delegate: LoginDelegate? //delegate to notify our presenting controller we logged in
    
    //the login button action
    @IBAction func login(sender: UIButton) {
        loginButton.enabled = false //disable the button while we attempt the login
        createButton.enabled = false
        //new we call the User.login method that we just reviewed
        User.login(nameField.text, password: passwordField.text, success: { (user: User) in
          //that returns our new User object and we pass that to our delegate
            self.delegate?.didLogin(user) 
            }, { (error: NSError) in
                self.errorFinished(error)
        })
    }
    //the create button action
    @IBAction func create(sender: UIButton) {
        loginButton.enabled = false
        createButton.enabled = false
        User.create(nameField.text, password: passwordField.text, imageUrl: "http://vluxe.io/assets/images/logo.png", //could be any imageUrl
            success: { (user: User) in
            self.delegate?.didLogin(user)
            }, { (error: NSError) in
                self.errorFinished(error)
        })
        
    }
    
    //show an alert and enable the buttons because the login or create failed
    func errorFinished(error: NSError) {
        println("unable to login")
        self.loginButton.enabled = true
        self.createButton.enabled = true
        let alert = UIAlertView(title: "Error", message: error.localizedDescription, delegate: nil, cancelButtonTitle: "Ok")
        alert.show()
    }
    
}
```

This all shows up in `MasterViewController.swift`

```swift
    override func viewDidAppear(animated: Bool) {
        super.viewDidAppear(animated)
        if user == nil {
            self.performSegueWithIdentifier("presentLogin", sender: self) //presentLogin is what we named our modal segue of the login controller 
        }
    }
```

I recommend checking out the full source for the client [here](https://github.com/Vluxe/Swift-API-Project). This includes the storyboard and all the code in context, which should hopefully clear up any ambiguity.

I am sure you notice there are a few Swift libraries we have in there to make API interaction easier, much like the Go libraries we use on the API side. They are managed with a simple package manager we created until CocoaPods supports Swift frameworks. The package manager name is named [Rouge](https://github.com/acmacalister/Rouge) and is written in Swift (Swift, installing Swift libraries!?!? Queue the inception horn). I do stress that it is _simple_ and is just a way to avoid the annoyance of git submodules. It is in no way the caliber or scale of CocoaPods, but will hopefully easy the burden for the time being. The Swift libraries used are listed at the end of the article for your reviewing pleasure.

Alright, that was a lot of code to digest between the server and client, but let's break down how much we have. We have a basic, yet fully functional authenticated API in Go. We also have the start of a fully functional iOS app in Swift. This gives a great starting point for creating both a Go API server and new app in Swift. If we tossed in JS front-end framework we will have all the parts for baseline and modern application (We only do the bleeding edge around here!). Next week we will finish off the client and show how to work use CRUD with our guitars.

As always, comments, questions, and random rants are appreciated.

[Twitter](https://twitter.com/daltoniam)

[SwiftHTTP](https://github.com/daltoniam/SwiftHTTP)

[Skeets](https://github.com/daltoniam/Skeets)

[JSONJoy](https://github.com/daltoniam/JSONJoy-Swift)

[Rouge](https://github.com/acmacalister/Rouge)

[Swift Project](https://github.com/Vluxe/Swift-API-Project)
