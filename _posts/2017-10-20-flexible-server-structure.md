---
layout: post
title:  "Superior Server Structs in Go"
date:   2017-10-02 12:30:00
categories: 'austin'
tags: Go
keywords: golang http server structs design patterns
cover: 'assets/images/server-structure-cover.jpg'
navigation: True
---


Hey gang, sorry we haven't talked in such a long time. Life got full, burnout is real and these articles fell by the wayside. The upshot is we are back, rested, refreshed with some wonderful content I think you are really going to love. I was recently working on a side project that got me thinking for the topic of this article. In the Go community we generally see this approach to structuring our HTTP applications:

```go
type HelloHandler struct {
    db *sql.DB
}
func (h *HelloHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    var name string
    // Execute the query.
    row := h.db.QueryRow(“SELECT myname FROM mytable”)
    if err := row.Scan(&name); err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    // Write it back to the client.
    fmt.Fprintf(w, “hi %s!\n”, name)
}
```

In case you have seen this code before, I actually borrowed it from Ben Johnson excellent Medium [article](https://medium.com/@benbjohnson/structuring-applications-in-go-3b04be4ff091) on how he structured his Go applications. It was actually the article that first introduced me to this design pattern and one I have been faithfully for my tenure of Go. This design has several advantages with probably the most prominent being the removal of global state and an easy way to access shared resources across the application. While I love how easy it is for my to wrap my head around this structure, something always bothered me about it. In all my applications, it becomes far too easy to start having things look like this:

```go
type Server struct {
	db   *sqlx.DB
	m    *mgo.Session
}

func main() {
	db, err := sqlx.Connect("postgres", os.Getenv("DATABASE_URL"))
	if err != nil {
		log.Fatal("FAILED TO INIT DB. PLEASE set env vars. :", err)
	}

	m, err := mgo.Dial(os.Getenv("MONGO_URL"))
	if err != nil {
		log.Fatal("FAILED TO INIT MONGO. PLEASE set env vars. :", err)
	}

	s := &Server{db: db, m: m}

    // Register our handler.
    http.HandleFunc("/hello", s.HelloHandler)
    http.HandleFunc("/hi", s.HiHandler)
    http.HandleFunc("/hallo", s.HalloHandler)
    http.HandleFunc("/hola", s.HolaHandler)
    http.ListenAndServe(":8080", nil)
}

func (s *Server) HelloHandler(w http.ResponseWriter, r *http.Request) {
	// use s to access shared resources...
	fmt.Fprintf(w, "hello!")
}

func (s *Server) HiHandler(w http.ResponseWriter, r *http.Request) {
	// use s to access shared resources...
	fmt.Fprintf(w, "hi!")
}

func (s *Server) HalloHandler(w http.ResponseWriter, r *http.Request) {
	// use s to access shared resources...
	fmt.Fprintf(w, "hallo!")
}

func (s *Server) HolaHandler(w http.ResponseWriter, r *http.Request) {
	// use s to access shared resources...
	fmt.Fprintf(w, "hola!")
}
``` 

Now I admit we don't have to design things this way and could avoid this problem if we created additional "Server/Handler" types, but too can get a little redundant. Here is my suggestion to avoid this:

```go
type Server struct {
	db   *sqlx.DB
	m    *mgo.Session
}

func main() {
	db, err := sqlx.Connect("postgres", os.Getenv("DATABASE_URL"))
	if err != nil {
		log.Fatal("FAILED TO INIT DB. PLEASE set env vars. :", err)
	}

	m, err := mgo.Dial(os.Getenv("MONGO_URL"))
	if err != nil {
		log.Fatal("FAILED TO INIT MONGO. PLEASE set env vars. :", err)
	}

	s := &Server{db: db, m: m}

    // Register our handler.
    http.Handle("/hello", HelloHandler(s))
    http.Handle("/hi", HiHandler(s))
    http.Handle("/hallo", HalloHandler(s))
    http.Handle("/hola", HolaHandler(s))
    http.ListenAndServe(":8080", nil)
}

func HelloHandler(s *Server) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// use s to access shared resources...
		fmt.Fprintf(w, "hello!")
	})
}

func HiHandler(s *Server) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// use s to access shared resources...
		fmt.Fprintf(w, "hi!")
	})
}

func HalloHandler(s *Server) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// use s to access shared resources...
		fmt.Fprintf(w, "hallo!")
	})
}

func HolaHandler(s *Server) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// use s to access shared resources...
		fmt.Fprintf(w, "hola!")
	})
}
```

Here is what I like about this approach. Just like with our earlier structure, we are still avoid global state. While I think arguably this is a little bit tougher to reason through the extra layer of `http.HanderFunc`s, I feel the flexibility is worth it. We no longer have all the handlers tied to `struct` with all of our shared resources. We have the freedom to inject whatever parameters we would like. For example, let's say some of the handlers need to access an ElasticSearch server, but we didn't want to necessarily add that to one of our server types. In the initial app structure, we would have to create another will all of the resources the HTTP handler would need access to. With this approach, you could simply just pass along the ElasticSearch connection as another parameter and you are off to the races. I think probably the most powerful part of this design is it opens up the ability to use interfaces. For example:

```go
type Server interface {
	LegitQuery(s string) string
}

type DB struct {
	db   *sqlx.DB
}

type Remote struct {
	test string
}

func main() {
	db, err := sqlx.Connect("postgres", os.Getenv("DATABASE_URL"))
	if err != nil {
		log.Fatal("FAILED TO INIT DB. PLEASE set env vars. :", err)
	}

	d := &DB{db: db}
	r := &Remote{test: "remote resources!!"}

    // Register our handler.
    http.Handle("/hello", HelloHandler(d))
    http.Handle("/hi", HiHandler(r))
    http.ListenAndServe(":8080", nil)
}

func HelloHandler(s Server) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// use s to access shared resources...
		// s.LegitQuery("Go is great!")
		fmt.Fprintf(w, "hello!")
	})
}

func HiHandler(s Server) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// use s to access shared resources...
		// s.LegitQuery("Go is great!")
		fmt.Fprintf(w, "hi!")
	})
}

func (d *DB) LegitQuery(s string) string {
	// do some db query
	return ""
}

func (r *Remote) LegitQuery(s string) string {
	// do some remote query
	return ""
}
```

As see above, we could easily switch out the implementation of our `Server` to be a local database, remote call like gRPC or whatever we like and nothing has to change with our HTTP handlers. This also works great for testing, as we can create mock Server objects to easily exercise our handlers. In the end, I think this application structure for our Go programs gives you a great balance between easy of use and flexibility. As always, I would be elated to hear y'all thoughts on this and if you like what you saw, I'm currently on the market for some new opportunities. Feel free to hit me up on [twitter](https://www.twitter.com/acmacalister) or shoot me an [email](mailto:acmacalister@gmail.com).

- [Structuring Applications in Go](https://medium.com/@benbjohnson/structuring-applications-in-go-3b04be4ff091)