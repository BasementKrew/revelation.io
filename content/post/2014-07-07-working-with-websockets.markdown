---
title:  "Working With Websockets"
date:   2014-07-07T10:00:00-07:00
author: "<a href='http://daltoniam.com'>Dalton Cherry</a>"
author_image: "http://www.gravatar.com/avatar/2fdc7b889f35118a7334187b15c5b957.png?r=x&amp;s=320"
summary: "Despite all the Swift we have been having, we haven't forsaken Objective-C. This week we break down the websocket protocol by exploring the newest Objective-C websocket library, jetfire."
tags: Objective-C, objc, apple, cocoa, websocket, osx, ios
---

The world needs websockets! The rise of realtime applications on the web, desktop, and mobile all but demands it. Ok, that might be a bit theatric, but certainly we can all agree that there has been a significant growth in realtime needs in the last 5 years. The websocket protocol answers this call and is now the supported standard in all major web browsers. This is huge, as it means all platforms can do realtime over the same protocol. Let's jump in!

The most powerful part of websockets is how easily it inter opts with HTTP. Establishing a websocket actually starts with a HTTP request.

```
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
 Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
Origin: http://example.com
```

This works like a normal HTTP request. A TCP connection is opened and the HTTP payload is sent, but unlike a normal HTTP request which would return a response and disconnect the client, the connection stays open. The interesting part here is we don't use our standard `NSURLSession` to do this HTTP request as that expects the typical HTTP half duplex model of send -> response -> close. We use the lower level `CFStreamCreatePairWithSocketToHost` (which can be replace by `NSStream getStreamsToHostWithName:port:inputStream:outputStream:` on iOS 8 and OSX 10.10) to establish the connection. This just opens a raw TCP connection and we use the Foundation APIs to create and send the HTTP request over the raw socket.

```objc
CFURLRef url = CFURLCreateWithString(kCFAllocatorDefault, (CFStringRef)self.url.absoluteString, NULL);
CFStringRef requestMethod = CFSTR("GET");
CFHTTPMessageRef urlRequest = CFHTTPMessageCreateRequest(kCFAllocatorDefault,
                                                         requestMethod,
                                                         url,
                                                         kCFHTTPVersion1_1);
CFHTTPMessageSetHeaderFieldValue(urlRequest,
                                 (__bridge CFStringRef)headerWSUpgradeName,
                                 (__bridge CFStringRef)headerWSUpgradeValue);
CFHTTPMessageSetHeaderFieldValue(urlRequest,
                                 (__bridge CFStringRef)headerWSConnectionName,
                                 (__bridge CFStringRef)headerWSConnectionValue);
CFHTTPMessageSetHeaderFieldValue(urlRequest,
                                 (__bridge CFStringRef)headerWSProtocolName,
                                 (__bridge CFStringRef)headerWSProtocolValue);
CFHTTPMessageSetHeaderFieldValue(urlRequest,
                                 (__bridge CFStringRef)headerWSVersionName,
                                 (__bridge CFStringRef)headerWSVersionValue);
CFHTTPMessageSetHeaderFieldValue(urlRequest,
                                 (__bridge CFStringRef)headerWSKeyName,
                                 (__bridge CFStringRef)[self generateWebSocketKey]);
CFHTTPMessageSetHeaderFieldValue(urlRequest,
                                 (__bridge CFStringRef)headerWSKeyName,
                                 (__bridge CFStringRef)[self generateWebSocketKey]);
CFHTTPMessageSetHeaderFieldValue(urlRequest,
                                 (__bridge CFStringRef)headerOriginName,
                                 (__bridge CFStringRef)self.url.absoluteString);
CFHTTPMessageSetHeaderFieldValue(urlRequest,
                                 (__bridge CFStringRef)headerWSHostName,
                                 (__bridge CFStringRef)[NSString stringWithFormat:@"%@:%@",self.url.host,self.url.port]);

NSData *serializedRequest = (__bridge NSData *)(CFHTTPMessageCopySerializedMessage(urlRequest));
CFReadStreamRef readStream = NULL;
CFWriteStreamRef writeStream = NULL;
CFStreamCreatePairWithSocketToHost(NULL, (__bridge CFStringRef)self.url.host, [self.url.port intValue], &readStream, &writeStream);

self.inputStream = (__bridge_transfer NSInputStream *)readStream;
self.inputStream.delegate = self;
self.outputStream = (__bridge_transfer NSOutputStream *)writeStream;
self.outputStream.delegate = self;
[self.inputStream scheduleInRunLoop:[NSRunLoop currentRunLoop] forMode:NSDefaultRunLoopMode];
[self.outputStream scheduleInRunLoop:[NSRunLoop currentRunLoop] forMode:NSDefaultRunLoopMode];
[self.inputStream open];
[self.outputStream open];
[self.outputStream write:[data bytes] maxLength:[data length]];
self.isRunLoop = YES;
while (self.isRunLoop)
    [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
```

That is a fair bit of code, but should be pretty easy to follow. It starts by forming up an HTTP request and supplying the required headers. The HTTP request is then written over the output stream. Looking at the code above, we can appreciate the simplicity that `NSURLSession` provides in sending HTTP requests, but this is necessary for websockets as we need access to underlining TCP socket.

Now that we connected and requested to become a websocket the server will hopefully respond with something like this:

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: HSmrc0sMlYUkAGmm5OPpG2HaGWk=
Sec-WebSocket-Protocol: chat
```

This means we now have a websocket that is a long lived TCP socket to our server which makes realtime possible by using the bi-directional nature of a TCP socket. This all happened over HTTP so we can use the standard HTTP ports of 80 or 443, which is a big deal, as it removes any problems we might have with firewalls blocking non standard ports. If the websocket protocol just stopped here, it would still be a huge win, but it goes even further and provides a really clean way to work with the fragmented nature of packets called **The Base Framing Protocol**.


The base framing protocol is built to be a thin layer over TCP to handle the needs of fragment data. It is built like so:

```
0                   1                   2                   3
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
|     Extended payload length continued, if payload len == 127  |
+ - - - - - - - - - - - - - - - +-------------------------------+
|                               |Masking-key, if MASK set to 1  |
+-------------------------------+-------------------------------+
| Masking-key (continued)       |          Payload Data         |
+-------------------------------- - - - - - - - - - - - - - - - +
:                     Payload Data continued ...                :
+ - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
|                     Payload Data continued ...                |
+---------------------------------------------------------------+
```

The RFC covers in detail which each bit is for, but I just want to touch on the few main ones.

- **FIN** The FIN bit notifies if this is the last packet in a fragment packet message. This way the client knows when a full message has been received.

- **opcode** The opcode is an `int` and tells the client the type of message this is. Examples of this would be the message payload in binary or text, if it is a continue frame of a previous message or is a ping to check if the client is still alive.

- **Payload Length** The payload length is interesting as it can mean a few different things. If the length is 125 or below, that is the actual length of the payload data. If the length is 126 or a 127 it means that the next 2 or 8 bytes are going to be used for the payload length. This allows some pretty huge packets to be sent, but this allows quite a bit of flexibility in the protocol.

It is important to note that while the base framing protocol provides a smart and simple way to handle message fragments, it is not a magic bullet for the overall fragmentation of raw TCP socket reading and writing. The client library must still handle this and ensure the message it is processing is a full message, else erroneous results may (will) occur.

Overall websockets are a simple, yet powerful protocol to do realtime communication. The simplicity, interoperability, and availability is going to make this the obvious choice for real time communication of today as well as the future. I am excited to see what kind of apps may adopt the websocket protocol as the base of their realtime solution and possible phase out the numerous HTTP APIs of different services. As always, questions, comments, feedback, and random rants are appreciated.

[jetfire](https://github.com/acmacalister/jetfire)

[RFC 6455](http://tools.ietf.org/html/rfc6455)

[Twitter](https://twitter.com/daltoniam)



