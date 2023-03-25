---
date: 2023-03-24
categories:
- Personal project
- NodeJS
- uWebSockets.js
- Dev
links:
- GitHub repository: https://github.com/jordan-breton/uws-reverse-proxy
- NPM package: https://www.npmjs.com/package/uws-reverse-proxy
---

![uws-reverse-proxy: let's reconcile uWebSockets.js and the NodeJS ecosystem](/assets/images/blog/uws-reverse-proxy-lets-reconcile-uwebsockets-js-with-the-nodejs-ecosystem/cover.jpg){ .cover }

# **uws-reverse-proxy**: let's reconcile **uWebSockets.js** and the **NodeJS ecosystem**

[uws-reverse-proxy](https://github.com/jordan-breton/uws-reverse-proxy) is an **easy-to-use 0-dependency**\* reverse proxy based on [uWebSockets.js](https://github.com/uNetworking/uWebSockets.js/)
I created to solve a problem I encountered recently for one of my clients.

This package enables use of `uWebSockets.js` and any **HTTP** server on **the same port** in a matter of seconds and fewer than
ten lines of code :wink:

<!-- more -->

!!! warning

    To get a better context about the genesis of the [**uws-reverse-proxy** package](https://github.com/jordan-breton/uws-reverse-proxy){ target="_blank" } and the problems it solves,
    please take few minutes to read [uWebSockets.js: the package that deserves a greater popularity](/blog/2023/03/23/uwebsockets-js-the-package-that-deserves-a-greater-popularity/){ target="_blank" }

## What is `uws-reverse-proxy` ?

A... reverse proxy based on [uWebSockets.js](https://github.com/uNetworking/uWebSockets.js). Now that we made things so much more obvious,
the questions you may ask yourself are: 

!!! question "a) what is a proxy?"

    The quick answer: it's an in an intermediate between a client and a server.

    More specifically, a proxy is meant to protect a client from the Internet by guaranteeing its privacy.

    For both **proxies** and **reverse proxies**, the network architecture stay identical:
    
    === "With proxy"
    
        ```mermaid
        sequenceDiagram
            participant C as Client
            participant P as Proxy
            participant S as Server
            C->>P: HTTP request
            activate P
            P->>S: Forwarded HTTP request
            activate S
            S->>P: HTTP Response
            deactivate S
            P->>C: Forwared HTTP response
            deactivate P
        ```
    
    === "Without proxy"
    
        ```mermaid
        sequenceDiagram
            participant C as Client
            participant S as Server
            C->>S: HTTP request
            activate S
            S->>C: HTTP Response
            deactivate S
        ```

!!! question "b) Okay... but, what is a ^^reverse^^ proxy?"

    While a **proxy** is meant to protect the **client**, a **reverse proxy** is meant to protect the **server**.

    When you have a proxy, it doesn't know in advance what kind of website/service you're going to reach: it only
    protects you by acting in your behalf.

    The reverse proxy, in another hand, is configured to forward to one or more server configured in advance. It acts on 
    the behalf of the server, hiding it from you.

The goal of **uws-reverse-proxy** is to help **uWebSockets.js** users to work... with **uWebSockets.js**, because as
excellent and performant this library is, its main drawback is its lack of interoperability with the rest of the **NodeJS**
ecosystem, especially if you want to combine it with an [Express](https://expressjs.com/fr/) or [NestJS](https://nestjs.com/) server.

As many projects **require** that their user can access the service they provide from everywhere, you need 
your application to accept connection on the port **443**, since it's the only one that is open in restricted networks like
enterprise/school networks.

And that's not to mention some server environments, like clouds, that just provide you only **one** open port and
don't allow you to install or configure any proxy.

If you're interested in this topic, I encourage you, again, to read [uWebSockets.js: the package that deserves a greater popularity](/blog/2023/03/23/uwebsockets-js-the-package-that-deserves-a-greater-popularity/){ target="_blank" }
if you didn't yet.

## Installation & usage

Let's start by installing it in your project:

With npm:

```bash
npm install uws-reverse-proxy
```
{ no-linenums }

With yarn:

```bash
yarn add uws-reverse-proxy
```
{ no-linenums }

Next, we'll say that you provide an **uWebSockets.js:SSLApp** instance in `uwsApp`, and the only code you'll have to write is:

```js
const {
	UWSProxy,
	createUWSConfig,
	createHTTPConfig
} = require('uws-reverse-proxy');

const proxy = new UWSProxy(
	createUWSConfig(uwsApp /* (1)! */, { port: 443 /* (2)! */ }),
	createHTTPConfig({
        host: '127.0.0.1', // (3)!
        port: 8080, // (4)!
        protocol: 'http' // (5)!
    })
);

proxy.start();
```

1.  uWebSockets.js [SSLApp](https://unetworking.github.io/uWebSockets.js/generated/functions/SSLApp.html){ target="_blank" }
2.  The public port you want your application to accessible on. Here, it is **443** (https)
3.  The listening **host** for your HTTP server (Express, NestJS, Koa, whatever...). If the proxy and the http server are on the same machine, omit this parameter, it will default to **127.0.0.1** anyway.
4.  The listening **port** for your HTTP server (Express, NestJS, Koa, whatever...). If the proxy and the http server are on the same machine, omit this parameter, it will default to **35794**.
5.  The **protocol** used by your HTTP server (Express, NestJS, Koa, whatever...). If the proxy and the http server are on the same machine, omit this parameter, it will default to **http**. If they're not, 
    and if the proxy and the HTTP server are **not in the same physical or virtual network**, you **must** enable https to secure traffic between the proxy and the server.

??? example "See a more complete example using **Express**:"

    In this example, `UWSProxy` will create itself an **uWebSockets.js** `App` (not an `SSLApp`, for brevity/clarity concerns). 

    ```js
    const http = require('http');
    const express = require('express');
    const uWebSockets = require('uWebSockets.js');
    
    const {
        UWSProxy,
        createUWSConfig,
        createHTTPConfig
    } = require('uws-reverse-proxy');
    
    const port = process.env.PORT || 80;
    
    const proxy = new UWSProxy(
        createUWSConfig(
            uWebSockets,
            { port }
        )
    );
    
    const expressApp = express();
    expressApp.listen(
        proxy.http.port,
        proxy.http.host,
        () => console.log(`Express Server listening at ${proxy.http.protocol}://${proxy.http.host}:${proxy.http.port}`)
    );
    
    proxy.uws.server.ws({
        idleTimeout: 10,

		open(){  console.log('New client connected!') },
		close(){ console.log('Client disconnected!')  },

		message(socket, data, isBinary){
			if(isBinary) socket.send("Sorry, I don't support binary :/");
			else{
				const msg = TextDecoder.decode(data);

				console.log('Message received: ', msg);

				if(msg === 'ping'){
					socket.send('pong - ' + (new Date()).toUTCString());
				}else{
					socket.send('Sorry, I only play ping-pong :(');
				}
			}
		}
    });
    
    proxy.uws.server.listen('0.0.0.0', port, listening => {
        if(listening){
            console.log(`uWebSockets.js listening on port 0.0.0.0:${port}`);
        }else{
            console.error(`Unable to listen on port 0.0.0.0:${port}!`);
        }
    });
    ```

If you want to play further, you should take a look at the demo repository I created. It even provide a standalone
proxy mode, for you to test **uws-reverse-proxy** with your own HTTP server without changing your code.

<div class="center" markdown>

[:material-flask: Try the demo](https://github.com/jordan-breton/uws-reverse-proxy-examples){ target="_blank" .md-button .md-button--primary }

</div>

And you'll find below the links to the github repository and the npm package. Feels free to 
contribute and/or report any issue :wink:

<div class="center" markdown>

[:fontawesome-brands-github: GitHub](https://github.com/jordan-breton/uws-reverse-proxy){ .md-button .md-button--primary }
[:simple-npm: NPM](https://www.npmjs.com/package/uws-reverse-proxy){ .md-button }

</div>
    