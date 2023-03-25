---
date: 2023-03-23
categories:
- NodeJS
- uWebSockets.js
- Dev
---

![uWebSockets.js: the package that deserves a greater popularity cover](/assets/images/blog/uwebsockets-js-the-package-that-deserves-a-greater-popularity/cover.jpg){ .cover }

# uWebSockets.js: the package that deserves a greater popularity

When it comes to websockets the first package name that pop out of our mind in the **NodeJS** ecosystem is... [Socket.IO](https://socket.io/).
I would be very surprised if you were surprised!

The thing is that **Socket.IO** is one of the less efficient solutions when it comes to performances. The different wrappers
it's based upon just add a tremendous overhead to websocket handling, and still has opaque to non-existent backpressure management as it can be seen
in issues [#3158](https://github.com/socketio/socket.io/issues/3158){ target="_blank" } and [#4435](https://github.com/socketio/socket.io/issues/4435){ target="_blank" }.

Many other alternatives exist, but when performances start to be a major concern, you must find something more efficient.
That's why I would want to introduce you to uWebSocket.js!

<!-- more -->

## uWebSockets.js

Long story short, [uWebSockets.js](https://github.com/uNetworking/uWebSockets.js/) is an **efficiency** focused library written in **C++**.
Did you ever ask yourself if your raspberry PI 4 could handle **100k** simultaneous **secure** websockets connections sending messages every two seconds?

With uWebsockets.js, it will handle it pretty smoothly (as long as you don't try to handle compute intensive tasks, obviously), where socket.IO will start 
to be very unstable in about **10k** simultaneous connections[^1].

[^1]: According to [100k secure WebSockets with Raspberry Pi 4](https://unetworkingab.medium.com/100k-secure-websockets-with-raspberry-pi-4-1ba5d2127a23), by the maintainer himself.

The point of this example is not that much to prove that it can handle thousands concurrent connections, but rather to show how efficiently it does. Because the less time is spent in I/O writes and read,
the more time **our** apps have to do their work.

Sadly, as everything, it comes with a tradeoff: if uWebSockets.js succeed in being utterly efficient in what it does, it fails on one point. A very important one.

## Interoperability

Most (if not the vast majority) of the **NodeJS**'s ecosystem's webservers relies on [`node:http`](https://nodejs.org/docs/latest-v18.x/api/http.html).
Chances are that your preferred webserver package is using it under the hood, among which we find **Express**, **Fastify**, **NestJS** and **Koa**.

All of them will let us configure a `node:http` server with their callback, or will even create it for us for convenience, and the same applies for most of
websockets packages out there: [ws](), [websocket-node](), [socket.IO]() and many more relies on the `node:http` module.

!!! info "Wait... we need an HTTP server to handle websockets?"

    For those who are not very familiar with the websocket protocol, It always starts with an HTTP request that is upgraded
    to a full-duplex connection. As such, any websocket server is, in fact, an HTTP server.

So, what are the implications of this?

Having your websockets and your website on the same port is something you don't even have to care about in most cases if
you're a happy **NodeJS** developer.

To better understand what I mean, let me tell you a story that I have been told too many times!

## Study case

Let's say we started our project with **Express** (http) + **socket.IO** (websockets) as it is a very (very) common combination. 
Few years later our app gained traction, we installed several nodes on cloud providers and **socket.IO** starts to reach its limits, making our app unstable.
It crashes every time we face a peak usage. We could continue to add nodes or to vertically scale our nodes, but our infrastructure costs would explode.

We do some research and find out that **socket.IO** is slow. Not only is it inefficient, but its footprint on I/O, RAM and processing time
is quite huge compared to other solutions. To optimise our app, that is really websocket intensive, we come to the conclusion that we must change our websockets server.

By comparing many packages, we figure out that **uWebSockets.js** is fast and will certainly boost our app capacity by a factor of ten. Quite cool!

## Migrating from Socket.IO to uWebSockets.js

We start our tests, find out that we must open two ports in our dev environment. Not a big deal, we do. One or two weeks later, we successfully 
replaced **Socket.IO** in our dev environment. We push our pre-release on the test environment in the cloud and... no websocket connection anymore:
<span color="red">`ERCONNREFUSED`</span> errors are filling up our browser console. 

A drop of cold sweat makes us shiver. What happened?

Our cloud infrastructure only let us **^^ONE^^ public port**. If it is possible, we upgrade our plan and set up our new port. If it is not... we're lucky
because we won't push a big misconception to production. But today we're not lucky: our cloud provider let us update our plans to open a new port. 

If you don't see it coming, wait for it :wink:

Our test passes. We push our release in production and...

Some clients start to call us out for support: they do not have any websocket connection anymore. After some investigations,
you finally discover where is the problem: they try to reach your app from within a restrictive **NAT** or behind a strict firewall.

So, all of your app traffic must be routed on the same port: **443**.

A new drop of cold sweat makes you shiver again: **uWebSockets.js** and `node:http` are two different servers! 
Since **uWebSockets.js** do not rely on any **NodeJS** networking tools, it will have to listen to its own port.

In simpler words: uWebSockets.js is **incompatible** with most of NodeJS's webservers.

**OUCH**

## How serious is it, doc?

Needing two different open ports didn't seem to be a big deal at first glance, isn't it? It was a wrong assumption.

### Network consideration

It is a big deal if any of your users need to access it from a restrictive **NAT** or behind a strict firewall. In those conditions, your client will only be able
to browse the internet through the port **443** (sometimes, port **80** too but as an insecure protocol its usage slowly fades away). It's usually the case for enterprise/school networks.

This is a well-known problem, and we already have solutions for it: just use a proxy like **Apache** or **Nginx**! The proxy will listen for all incoming requests and dispatch
them to the port you want based on any arbitrary criteria regarding the said requests.

Problem solved!

No?

In most cases, yes, it is. Until you try to deploy your app in the cloud, or until you allow your customers/users to use your app on premises.
In the first case, many cloud providers like [Heroku](https://www.heroku.com/) or [Render](https://render.com/) will only give you one **public** port
to listen to and won't let you install any proxy, and if they allow you to change some configuration, they're not likely to let you open
ports at will.

On premises, depending on your user's/customer's business environment, it may just cost too much to open a second port even if the company allows it, which is not
very likely, especially if it has a strict security policy. 

And yet, opening this second port is not a solution since some of our users will not be able to use our service anyway.

So... are we screwed ?

## Our options

To pretty much every problem a solution! Let's see what we can do :smile:

### 1) Setting up a dedicated proxy in front of our infrastructure

<div class="grid cards" markdown>

-    :material-check:{ .pros } **Pros**

     ---

     - Almost no change in the code (we may want to provide a proxy authentication mechanism)

-    :material-close:{ .cons } **Cons**

     ---

     - Still needs to upgrade cloud offers to open two ports.
     - More expensive
     - More maintenance and skills needed
     - Still impact performances[^2] and adds latency
     - Adds a point of failure in the network

</div>

[^2]:
      Any layer / new tenant adds its footprint and must be measured, especially in a scenario where we want something
      optimized.


### 2) Using an **express** proxy middleware to forward websockets to **uWebSockets.js**

<div class="grid cards" markdown>

-    :material-check:{ .pros } **Pros**

     ---

     - Easy to do
     - Almost no code impact
     - Almost no configuration
     - Feature embedded in code, do not depend on the host

-    :material-close:{ .cons } **Cons**

     ---

     - Defeat uWebSockets.js performances since `node:http` is slower/less efficient

</div>

### 3) Changing cloud provider for a more permissive one

<div class="grid cards" markdown>

-    :material-check:{ .pros } **Pros**

     ---

     - No code change
    
-    :material-close:{ .cons } **Cons**

     ---

     - Must move the whole infrastructure (code + data)
     - Must learn new cloud providers processes and pitfalls
     - Probably more expensive (especially if long-term subscriptions was purchased for the current infrastructure)

</div>

### 4) Going bare metal

<div class="grid cards" markdown>

-    :material-check:{ .pros } **Pros**

     ---

     - Total freedom
     - Reduces gross costs

-    :material-close:{ .cons } **Cons**

     ---

     - Must move the whole infrastructure (code + data)
     - Maintenance costs rise up
     - Requires more knowledge and skills to set up and troubleshoot
     - Requires even more knowledge and skills when scaling is needed
</div>

### 5) Dropping **express**

<div class="grid cards" markdown>

-    :material-check:{ .pros } **Pros**

     ---

     - Best performances, even for the HTTP server

-    :material-close:{ .cons } **Cons**

     ---

     - Costs+++ since it will need a huge app rewrite
     - Can't rely on any express middleware
     - Since uWebSockets.js is not widely used, it can be hard (or even impossible) to find a replacement for the ones we're using.
       That being said, the community behind uWebSockets.js tends to create express compatibility layers. But what if you use... NestJS? Koa? Fastify?

</div>

## Conclusion
    
Sadly, there is the conclusion: if we are pragmatic, we must either opt in for solution 2 (Using an _express_ proxy middleware 
to forward websockets to uWebSockets.js) that disqualify uWebSockets.js since it will never perform better than express. Or... using a less efficient websockets server that do better than **Socket.IO**
and dropping uWebSockets.js until we can afford a migration.

Quite depressing, isn't it?

But... what if I told you that there is a better option? (What suspense!)

What if we used **uWebSockets.js** as an HTTP proxy for all non-websocket traffic? We would have the best of both worlds:

- Best performances possible for websockets
- Very light impact on the HTTP server
- No code changes for the HTTP server, with all the power of the entire NodeJS ecosystem around it

It's why I created and published a package to solve this exact problem: **uws-reverse-proxy**. If you want to know more about it,
click on the button below :wink:

<div class="center" markdown>

[Discover uws-reverse-proxy](/blog/2023/03/24/uws-reverse-proxy-lets-reconcile-uwebsockets-js-with-the-nodejs-ecosystem/){ .md-button .md-button--primary } 

</div>