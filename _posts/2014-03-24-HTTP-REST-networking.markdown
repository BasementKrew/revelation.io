---
layout: post
title:  "HTTP and RESTful networking"
date:   2014-03-24 08:00:00
author: "<a href='http://daltoniam.com'>Dalton Cherry</a>"
summary: "Today we venture to dig down into the raw building blocks of HTTP and define what exactly RESTful means for a client side programmer."
tags: HTTP, REST, ruby, rails, GET, POST, PUT, DELETE
---

Using HTTP in a RESTful way. Arguable the most used design pattern and protocol in computer science today. Despite it's common use, have we ever taken the time to dig down and understand what is exactly happening under the hood?

Let's start by getting some terms defined. HTTP is defined as the "Hypertext Transfer Protocol". It is the core of how we communicate on the web today. In fact you had to use HTTP to read this article right now. REST is defined as "Representational state transfer". It is a design pattern for HTTP 1.1 and how to use it efficiently. REST is a way to use HTTP to maximize the use of the HTTP protocol. Before we dig to deep into REST let's step back and take a deeper look at HTTP.

HTTP is build on the TCP (Transmission Control Protocol). Without diving to deep into TCP (we will do that in a [week or two](TCP-networking.html)), it provides a reliable, ordered connection. This means it guarantees that we will receive or more specifically know, if we have received all the data sent. A great analogy of this is when buying items from a store (like groceries, a cup of noodles, or the latest Apple products). When you finish your purchase, you are given a receipt with an itemized breakdown of what you bought. This allows you to guarantee you got everything you purchased. The same thing exist in TCP. It makes sure we got all the data we requested. If not, we would be missing pieces of our web page (which might make this article tough to understand otherwise!)

Now that we got our bases covered, let's dig into the meat of HTTP. Here is an example of a common outgoing HTTP packet.

```
GET /index.html HTTP/1.1
Host: www.example.com
```

Pretty simple, eh? Most HTTP packets include a few more fields than this, but this is a very simple packet for demonstration purpose. It does well to convey the basics of a standard HTTP packet. Now you are probably wondering how TCP fits into all of this as well? Well have no fear, code blocks are here!

```
Ethernet Header
IP Header
TCP header
TCP Payload (which is our HTTP request from above)
```

In networking, we use a layered approached to build packets. To avoid going into to much detail again (for now), let's just settle that the headers build on each other in the packet. HTTP is in the payload of TCP which comprises the rest of the packet after the headers. This is a flexible and effective way to move data without it being overly complex.

Great. We now have a 10,000 foot view on how an HTTP packet is structured. Let us break down the simple example of our HTTP packet. The first part is the HTTP verb we are preforming. We will circle back around to that in a shortly. `/index.html` is the page or resource we are requesting (e.g. your browser requested `HTTP-REST-networking.html` to view this page!).

`HTTP/1.1` is the HTTP version we are using to make the request. Simply put, HTTP use to be on version 1.0 which did not give us quite the power of 1.1. With the introduction of HTTP/1.1, we got most of the RESTful pattern (thanks HTTP/1.1 team!). HTTP/1.1 has been the standard for a while now, so don't worry about backwards support or anything crazy like that. `Host` is a HTTP header field. It tells us which host to route our request to (again, your browser added `vluxe.io` as the Host field to get you here). It is worth mentioning that Host is a header field. There can be and usually is more header fields. The header fields follow the same format as Host, so adding new fields is a straight forward process.

The last part of the HTTP request worth mentioning is invisible in this example. HTTP carriage returns (new lines or the `\n`) use the alternate (read Windows) format of `\r\n`. As much as this is the bain of all text editor designers everywhere, it makes a lot of sense from a Unix stand point. `\r\n` is used after `HTTP/1.1` in our example and at the end of the Host fields example.com value. This allows HTTP to freely embed `\n` in the response without fear of effecting the HTTP packets overall format (Which is good). Here is a example with those added

```
GET /index.html HTTP/1.1\r\n
Host: www.example.com\r\n
```
 With that, we said we would circle back to the verbs and now we will. Onward to the glory of HTTP verbs!

 HTTP verbs are the heart of what makes REST useful. The word 'verb' is a great term choice to describe the flow of what is happening. It let's our web server know what action we intend to preform. This concept is leverage in frameworks such as Ruby on Rails or Django to make adding, deleting, and modifying database records simple. Below is a list of common verbs and what they are used for.

**GET**

Requests a representation of the specified resource. Requests using GET should only retrieve data and should have no other effect. This is the most common verb used and is how we view most web pages.

```
GET /index.html HTTP/1.1
Host: www.example.com
```

**HEAD**

Asks for the response identical to the one that would correspond to a GET request, but without the response body. This is useful for retrieving meta-information written in response headers, without having to transport the entire content. A common use for this verb would be the progressive loading of images that we see in many apps today. The application sends a HEAD request to the image resource then reads the size of the image from the header. It subsequently sends a GET request and compares the size of the data it gets back in the several packets from the GET response to the total size it already has. Then it does some simple math magic to know how much is left and viola, pretty loading dialogs for everyone!

```
HEAD /index.html HTTP/1.1
Host: www.example.com
```

**POST**

Requests that the server accept the entity enclosed in the request body as a new subordinate of the web resource identified by the URI. In normal human words, this means it sends a set of data fields to the server and expects the server to do something with them. This would normally include creating some kind of record, like a database value, such as in Rails or Django. There are 2 types of POST request. One is `application/x-www-form-urlencoded` which is used to send form values of a textual nature. On the other hand there is `multipart/form-data`, which is used to upload files. The format of each is very different so use discretion when creating these.

```
POST /create HTTP/1.1
content-type:application/x-www-form-urlencoded;charset=utf-8
Host: www.example.com
content-length:207

paramValue=someValueToPost&someOtherParamValue=SomeOtherValue
```

```
POST /create HTTP/1.1
content-type:multipart/form-data, boundary=AcB03x;
Host: www.example.com
content-length:207

--AcB03x
content-disposition: form-data; name="field1"

$field1
--AcB03x
content-disposition: form-data; name="field2"

$field2
--AcB03x
content-disposition: form-data; name="postvalueName"; filename="theFilesName"
Content-Type: FileMimeType
Content-Transfer-Encoding: binary

binary (file) data here....
--AcB03x--
```
**Note:** The boundary is randomly generate value, we are just using `AcB03x` in this example.

**PUT**

Requests that the enclosed entity be stored under the supplied URI. Once again to put it in normal human vernacular, This usually refers to updating an existing record which again, is something like a database value. It is possible to use this to create a value if one does not exist, basically mimicking what a POST verb does.

```
PUT /resource HTTP/1.1
content-type:application/x-www-form-urlencoded;charset=utf-8
Host: www.example.com
content-length:207

paramValue=someValueToPost&someOtherParamValue=SomeOtherValue
```

```
PUT /resource HTTP/1.1
content-type:multipart/form-data, boundary=AcB03x;
Host: www.example.com
content-length:207

--AcB03x
content-disposition: form-data; name="field1"

$field1
--AcB03x
content-disposition: form-data; name="field2"

$field2
--AcB03x
content-disposition: form-data; name="postvalueName"; filename="theFilesName"
Content-Type: FileMimeType
Content-Transfer-Encoding: binary

binary (file) data here....
--AcB03x--
```

**DELETE**

Removes the specified resource. Pretty straight forward. The record goes bye, bye.

```
DELETE /index.html HTTP/1.1
Host: www.example.com
```

Alright, that was a lot to digest, but we now know how to boss our web server around. It is important to note there are a couple of other verbs, but the ones listed are the most common. Now I am sure you having been thinking, "this is all great pearls of wisdom, but what about the response, man?" (I know, I am basically a mind reader). Fair enough, up to this point we have only been concerned with our actions, now let's take a look at our responses.

The good news is the different verbs are pretty consist in there responses. Here is an example response

```
HTTP/1.1 200 OK
Date: Mon, 23 May 2005 22:38:34 GMT
Server: Apache/1.3.3.7 (Unix) (Red-Hat/Linux)
Last-Modified: Wed, 08 Jan 2003 23:11:55 GMT
Content-Type: text/html; charset=UTF-8
Content-Length: 131
Connection: close

<html>
<head>
  <title>An Example Page</title>
</head>
<body>
  Hello World, this is a very simple HTML document.
</body>
</html>
```
We already know what HTTP/1.1 is about. `200` is the HTTP status code of request. This is the server's way of telling the client if everything went as expected. A general rule of thumb for the status codes are: 2XX means the action was successful. 3XX means the request is redirecting somewhere else. 4XX means the request was not accessible (did not exist, does not have permissions, etc). 5XX means the server had an error, and the request was not successful.

We know already know what values below that are. They are headers values, just like the send request. They also use the same `\r\n` just like the send (hurray for consistency!). After the headers, the rest of the content is the HTTP payload which is the content we will do something with. For this example, we would display the HTML for this page.

In closing, this is not an exhaustive list of all the things HTTP can do. HTTP has been extended and is a constantly evolving for new demands of applications. Hopefully this gives you a base of how HTTP works and what is involved in creating a HTTP request (and maybe some more appreciation for the teams that make all this into some really simple APIs!).

For my current academic ventures, I am working on create a fully native OS X packet capture app called [Orca](https://github.com/Vluxe/Orca). I plan to continue to write about different networking related subjects as I continue my journey in the development of this app.

As always, questions, feedback, and comments are appreciated. [@daltoniam](http://twitter.com/daltoniam).