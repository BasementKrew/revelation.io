---
layout: post
title:  "Gopher Go! - Flag"
date:   2015-02-23 08:00:00
author: "<a href='http://austincherry.me'>Austin Cherry</a>"
author_image: "http://www.gravatar.com/avatar/4278893e11f873d60fede435f1ae08aa.png?r=x&amp;s=320"
summary: "Picking up where we left off in our golang series, today we will be reviewing the flag package."
tags: Go, golang, packages, net/http, flag, flags, unix, gnu
---

Flags are a well rooted Unix tradition. Chances are if you have been programming on a Unix based platform for any length of time you have had to use flags to configure behavior of different command line utilities. For those with a background in C programming you know how tedious and time consuming it can be to properly parse flags. Luckily in Go, we have a simple package in the standard library to do this for us.

```go
package main

import (
  "encoding/json"
  "flag"
  "fmt"
  "net/http"
  "net/url"
)

type condition struct {
  Temp string `json:"temp"`
  Text string `json:"text"`
}

type weather struct {
  Query struct {
    Results struct {
      Channel struct {
        Item struct {
          Condition condition `json:"condition"`
        } `json:"item"`
      } `json:"channel"`
    } `json:"results"`
  } `json:"query"`
}

var yahooWeatherUrl = "https://query.yahooapis.com/v1/public/yql?"
var city = flag.String("city", "Bakersfield", "Set the city. Default is Bakersfield.")
var state = flag.String("state", "CA", "Set the state. Default is CA.")

func main() {
  flag.Parse() // need to run this before using our flags.
  fmt.Println(flag.Args()) // print the rest of the flag args.
  v := url.Values{}
  v.Set("q", fmt.Sprintf("select item.condition from weather.forecast where woeid in (select woeid from geo.places(1) where text=\"%s,%s\")", *city, *state))
  v.Set("format", "json")
  v.Set("env", "store://datatables.org/alltableswithkeys")
  resp, err := http.Get(yahooWeatherUrl + v.Encode())
  if err != nil {
    fmt.Println(err)
  }

  var w weather
  decoder := json.NewDecoder(resp.Body)
  if err := decoder.Decode(&w); err != nil {
    fmt.Println(err)
  }
  c := w.Query.Results.Channel.Item.Condition
  fmt.Printf("The weather in %s, %s is %s with the temperature of %sF.\n", *city, *state, c.Text, c.Temp)
}
```

In case the code above is not immediately obvious, it is fetching the weather for the given city. If we run it without any flags we get something like this:

```
[]
The weather in Bakersfield, CA is Fair with the temperature of 63F.
```

See that we have two flags, `city` and `state` that except strings for their values.Notice also they have the default values of `Bakersfield` and `CA`, respectively. Interestingly enough, the flag values are actually pointers to the strings, which have to be deferenced to get their values. This holds true for pretty much every type supported in the flag package. If we run the program with the flags set, we get something like this:

```
./weather --city="New York" --state NY

[]
The weather in New York, NY is Fair with the temperature of 21F.
```

As the documentation states we can run the flags with either a single or double dash and include the equal sign or space like so:

```
./weather --city=Cupertino
./weather --city Cupertino
./weather -city Cupertino
```

A few things to note before we close. In the example above we are just using the default methods exported in the flag package. This is completely fine, but it is worth noting that you can create our own "custom" handling of flags using the `FlagSet` type in this package. It implements pretty much all the same methods and allows you to customize your usage method as well as the behavior for invalid flag parameters. One thing you may notice when using the flag package is how it does not behavior exactly like Unix or GNU based flags. According to Rob Pike as discussed in [this](https://groups.google.com/forum/#!topic/golang-nuts/3myLL-6mA94) go-nuts discussion this is by design. Meaning that things like this:

```
./weather --city=

[]
The weather in , CA is  with the temperature of F.
```

or this:

```
./weather --city="New York" other args ... --state NY

[other args ... --state NY] <-- other args that would have to be processed separately
The weather in New York, CA is Fair with the temperature of 21F.
```

That being said, there are a plethora of third party libraries that do more Unix/GNU style flag handling if that is more to your liking. As always, questions and comments are welcomed.

[Cobra](https://github.com/spf13/cobra)

[Kingpin](https://github.com/alecthomas/kingpin)

[Go Flags](https://github.com/jessevdk/go-flags)

[gnuflag](http://bazaar.launchpad.net/~gnuflag/gnuflag/trunk/files)

[flag](http://golang.org/pkg/flag/)