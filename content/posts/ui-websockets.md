---
title: "UI Websockets"
date: 2021-05-28T13:25:00
draft: false
authors:
  - Ian Bolton
tags:
  - crane
  - websockets
  - react
  - k8s
  - javascript
---

When creating a web client to consume a k8s api, you will need to make a decision on how to go about rapidly querying for changes.

#### Setup websocket server

To establish a websocket connection, a "handshake" between client and server is the place to start. The client will send an http request to the server which will then be upgraded to a websocket TCP connection if both the server and client support it.

Node server setup:

1. Setup an express server in the node app entry point.
2. Once the express server is configured, you can pass the express server instance to the webpack configuration. This allows both the express server & the websocket to share the same port.
3. Create the websocket configuration function & run it from within the node entrypoint.

   setupWebSocket.js - websocket configuration file

   ```js
   /** import axios which is an http client that works on both client and server **/
   const axios = require("axios");
   const port = process.env["EXPRESS_PORT"] || 9000;
   const k8s = require("@kubernetes/client-node");

   function setupWebSocket(app) {
     let WSServer = require("ws").Server;
     let server = require("http").createServer();
     let wss = new WSServer({
       server: server,
     });

     /** Forwards client side http requests to the existing express server **/
     server.on("request", app);

     wss.on("connection", (ctx) => {
       /** Catch all function that can be broken up to call other functions based on the JSON message type passed in via a websocket message event
        **/

       ctx.on("message", (data) => {
         let message;
         try {
           message = JSON.parse(data);
         } catch (e) {
           sendError(ws, "Wrong format");
           return;
         }
         /** Example to catch the GET_EVENTS type & perform a server side HTTP request to grab events using the k8s node client  **/
         if (message.type === "GET_EVENTS") {
           const kc = new k8s.KubeConfig();
           kc.loadFromDefault();
           const opts = {};
           kc.applyToRequest(opts);
           const eventsURL = `${
             kc.getCurrentCluster().server
           }/api/v1/namespaces/openshift-migration/events`;

           /** Since oauth flow is already implemented client side, passing the oauth token in the JSON was easy enough to do. This can be encrypted for production use. **/
           axios
             .get(eventsURL, {
               headers: {
                 Authorization: `Bearer ${message.token.access_token}`,
               },
             })
             .then(
               (response) => {
                 if (response.data) {
                   /** Compose a response JSON message to return to the client if the server side http request comes back OK **/

                   const messageObject = {
                     type: "GET_EVENTS",
                     data: response.data,
                   };
                   /** Server push a websocket message to the connected client **/
                   ctx.send(JSON.stringify(messageObject));
                 }
               },
               (error) => {
                 if (error) {
                   console.log(`error: ${error}`);
                 }
               }
             );
         }
       });

       ctx.on("close", () => {});
     });
     /** Listen for websocket connection attempts/messages & http requests to the express server on a specified port. **/
     server.listen(port, function () {
       console.log(`http/ws server listening on ${port}`);
     });
   }
   module.exports = setupWebSocket;
   ```

   main.js - node entrypoint

   ```js

   const express = require('express');
   const setupWebSocket = require('./setupWebSocket');
   const app = express();
   app.get("*", (req, res) => {
     res.render("index.ejs", { });
   });

   app.get('/login', async (req, res, next) => {
     /** Perform oauth flow here
     **/
   }
   setupWebSocket(app)


   ```

```


```

#### Setup websocket client

Once you have the websocket server configured, it is time to configure the client.

Within your existing react app, you can create a reusable hook to manage your websocket events. For demontstration purposes, I am using the library 'react-use-websocket' to handle connecting, reconnecting on unexpected disconnection, and sending/receiving messages.

1. Create a component that needs to communicate with your websocket server.

   ```jsx
   import React, { useState, useEffect } from "react";
   import useWebSocket, { ReadyState } from "react-use-websocket";

   export const WebSocketDemo = () => {
     const [socketUrl, setSocketUrl] = useState(
       `ws://127.0.0.1:${process.env.PORT || 9001}`
     );
     const messageHistory = useRef([]);
     const {
       sendMessage,
       sendJsonMessage,
       lastMessage,
       lastJsonMessage,
       readyState,
       getWebSocket,
     } = useWebSocket(socketUrl, {
       onOpen: () => console.log("opened"),
       //Will attempt to reconnect on all close events, such as server shutting down
       shouldReconnect: (closeEvent) => true,
     });

     const connectionStatus = {
       [ReadyState.CONNECTING]: "Connecting",
       [ReadyState.OPEN]: "Open",
       [ReadyState.CLOSING]: "Closing",
       [ReadyState.CLOSED]: "Closed",
       [ReadyState.UNINSTANTIATED]: "Uninstantiated",
     }[readyState];

     useEffect(() => {
       /** Grab the currentUser from localstorage to pass along with the websocket method for server side http authentication **/
       const LS_KEY_CURRENT_USER = "currentUser";
       const currentUser = JSON.parse(
         localStorage.getItem(LS_KEY_CURRENT_USER)
       );
       /** create a JSON message to send with the token & request type. The node server will use this to determine what type of http call to make **/
       const msg = {
         type: "GET_EVENTS",
         token: currentUser,
       };
       if (connectionStatus === "Open") {
         /** send the message to the open websocket. **/
         sendJsonMessage(msg);
       }
     }, [connectionStatus]);
     /** Render the last JSON message received from the server  **/
     return (
       <div>
         <div>The WebSocket is currently {connectionStatus}</div>
         {lastJsonMessage ? (
           <span>Last message: {lastJsonMessage?.data?.kind}</span>
         ) : null}
         <ul></ul>
       </div>
     );
   };
   ```
