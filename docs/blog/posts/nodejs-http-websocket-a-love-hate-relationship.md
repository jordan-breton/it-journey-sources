---
draft: true
date: 2023-02-26
categories:
  - NodeJS
  - NPM
  - Web
  - Programming
---

# NodeJS, HTTP and Websocket: a love/hate relationship

<!-- more -->

This combination is not as uncommon, as I encountered it several times. Typically, it happen in projects
that started with the combination **express** + **socket.io** or **ws**. As the project evolvs,
pressure on the WebSocket part intensifies.

    Sadly, **socket.io** is an innefficient tool that consume a lot of ressources. **uWebSocket**, in another hand,
    is the most performant websocket server available in the **NodeJS** ecosystem. 

    The thing is that **uWebSocket.js** and **express** are incompatible. 