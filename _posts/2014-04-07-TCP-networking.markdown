---
layout: post
title:  "Assembling into TCP"
date:   2014-04-07 08:00:00
author: "<a href='http://daltoniam.com'>Dalton Cherry</a>"
summary: "This week we continue our networking venture and unravel TCP."
tags: TCP, networking, packets, segments, HTTP
---

As promised from the [HTTP article](HTTP-REST-networking.html), we continue our digging into TCP. For a quick refresher, TCP stands for "Transmission Control Protocol". It is the core of networking done today and is by far the most popular protocol. Why is TCP so popular? Let's find out!

First off, let's start out with how a TCP Packet is structured:

```
Ethernet Header
IP Header
TCP header
TCP Payload (which can be whatever is needed, e.g. HTTP content)
```

When working with packets, most packet libraries (like libpcap) just return a `char *` buffer (for the readers that aren't C people, this means an array of some bytes). This means each header is literally aligned right next to the other one. At first this seems crazy. Why would anyone want to work with content like this? Packets are created in layers as we can see above. Ethernet builds the base, then IP, followed by TCP. We can easily swap out each layer for our needs (e.g. IPv4 or IPv6) and it does not change the way our buffer is returned to us. The important take away here is that packet headers are aligned in very specific ways to keep their size compact and flexible and TCP is no exception.

Now that we outlined a TCP packet's structure, let's go briefly over each layer of the packet. Ethernet is the base of the packet. This contains the source and destination mac addresses as well as the type of protocol that comes next (usually IPv4 or IPv6). The Ethernet header is a fixed size and always includes this data. Next comes the IP header. This varies on what it includes, depending on if it is IPv4 or IPv6. While the content is in different formats, it always includes a source and destination IP address, as well as a total size of the packet. It is important to note that both Ethernet and IP are considered "unreliable" protocols. This means the packets sent are not guaranteed to make it to their destination. TCP fixes this through some amazing wizardry that we outline below.

TCP is considered a "reliable" protocol. This means packets are guaranteed to make it to their destination. At this point your are probably wondering "wait, how can a reliable protocol be built off an unreliable one and guarantee that packets make it to their destination?". This is what makes TCP so unique. It does not actually have the power to force the packet to make it to its destination (that would be some true computer wizardry). Instead it verifies that packets arrive in order and are resent if lost in transmission. Think of it in this analogy. You go and buy some items from the store (like groceries, a cup of noodles, or the latest Apple products). When you finish your purchase, you are given a receipt with an itemized breakdown of what you bought. This allows you to guarantee you got everything you purchased and nothing was missed. The same thing exist in TCP. It has several fields to ensure that the packets sent are the ones received.

Alrighty, so now we have a high level view of how a TCP packet is built. Let's break down the TCP header itself.


```
srcPort
dstPort
sequence
acknowledgement
flags
offset
window
checksum
urgentPointer
payload
```

Boy oh boy, that is quite a few fields! Most of these fields are used for validation and what makes the packet reliable. We will go over each field briefly and highlight the most important details. If you are interested in reading a more in depth description of these fields I recommend checking out the wikipedia article on it [here](http://en.wikipedia.org/wiki/Transmission_Control_Protocol).

`srcPort` and `dstPort` are the application ports used. These ports are unique for every application. A well know example of this is port 80 and is used for most web traffic. While the client's (source) port varies, it's port out to the web server (destination) is always listening on port 80. This makes complete sense if you think about the server always listening on the same port so we always know how to get there. The client picks a random port to use for itself, which allows multiple port 80 requests to go out at the same time (this is really good, else loading stuff would be super slow).

Before we get into any of the other fields, it is important to know that TCP is "connection oriented". This means it has a direct handshake and exchange with a specific server. It sends a value to start the exchange called the handshake and expects the server to reply with a certain response. Once the handshake is establish, the content exchange can begin. The handshake is vital to ensuring that the validation is carried out properly. Now that we got that covered, onward and forward to the validation fields!



`sequence`, `acknowledgement`, `flags`, `offset`, `window`, `checksum`, `urgentPointer` are all fields used for validation. `sequence` is used to identify if this is the initial sequence number. TCP packets are broken up into sequences. This is because it is possible that not all the content can be sent in one packet, so the content is broken up into several different packets. Knowing if this is the first packet in the sequence makes sure we know the expected order of the packets.

`acknowledgement` is the next number in sequence if the ACK flag is set.

`flags` is a control bit to set the type or state of the packet. There are several different flags but the most notable are `SYN`, `ACK`, and `FIN`. `SYN` is used for a host of changes, but means to synchronize the sequence numbers. This is only set on the first packet handshake to establish the sequence number. `ACK` means that the sequence number has been acknowledge by the other host. `FIN` means no more content will be sent as we have reached the end.

`offset` specifies the size of the TCP header (since it can vary).

`window` is used for flow control. It specifics how large the sequence number can get.

`checksum` is use for error checking to ensure the packet has not been corrupt during transit.

`urgentPointer` is an offset from the sequence number when the `URG` flag is set. This field is used to process a packet out of order and make it's content "urgent".

`payload` is the content within the TCP (like our HTTP content to show this article!).

wow. That was a lot, but let's take a look at what we know now. We have successfully dissected a TCP packet and now have a basic understanding of why and how a TCP packet works. We worked mostly in the theoretical at this point and I strongly recommend running a packet capture to see a real life packet (WARNING, a plug for a personal project is imminent). This can of course be accomplished with my current academic undertaking [Orca](https://github.com/Vluxe/Orca). The first iteration of this will be released in the next few weeks and all feedback is welcomed.

As always, feedback, questions, and comments on this article are appreciated. Twitter: [@daltoniam](https://twitter.com/daltoniam).



