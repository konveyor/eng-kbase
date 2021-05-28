---
title: "UI Websockets"
date: 2021-05-21T13:43:26-04:00
draft: false
authors:
  - Ian Bolton
tags:
  - crane
  - state migration
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
3. Create the websocket configuration function & run it from the node endpoint.

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

Now assume you want to copy the following three files from the source PVC.

```
/mnt/pvc-0/10
/mnt/pvc-0/a/11
/mnt/pvc-0/a/51
```

Use the following steps:

1. On your local machine, create a temperory directory to store these files
   `mkdir ./pvc-files`
2. Use the following loop to copy the files over
   ```bash
   sudo oc login <server> -u <user> -p <password>
   sudo oc project source-namespace
   for i in /mnt/pvc-0/10 /mnt/pvc-0/a/11 /mnt/pvc-0/a/51; do
       sudo rsync --relative -a --progress --rsh='oc rsh' busybox-sleep:$i ./pvc-files/
   done
   ```
3. Verify the file `ls -lR ./pvc-files` permissions
4. Verify the files checksum
   ```
    for i in /mnt/pvc-0/10 /mnt/pvc-0/a/11 /mnt/pvc-0/a/51; do
           pod_checksum=$(oc exec busybox-sleep -- md5sum "$i" | awk '{printf $1}');
           local_checksum=$(sudo md5sum "/tmp/pvc-files$i" | awk '{printf $1}');
           if [[ $pod_checksum != $local_checksum ]]; then
                  echo "checksum for $i is not equal $pod_checksum $local_checksum"
           fi
    done
   ```
   if the for loop does not print anything all checksums are verified
5. Copy all the files to destination PVC. We will have to change directory
   into the pvc-0 folder to maintain the directory structure.

   ```bash
   sudo oc project destination-namespace
   cd pvc-files/mnt/pvc-0
   for i in 10 a/11 a/51; do
      sudo rsync -a --progress --relative --rsh='oc rsh' $i busybox-sleep:/mnt/pvc-0/  ;    done
   done
   ```

6. Verify the checksum from local to destination
   ```bash
   cd ../..
   oc project destination-namespace
   for i in /mnt/pvc-0/10 /mnt/pvc-0/a/11 /mnt/pvc-0/a/51; do
          pod_checksum=$(oc exec busybox-sleep -- md5sum "$i" | awk '{printf $1}');
          local_checksum=$(sudo md5sum "/tmp/pvc-files$i" | awk '{printf $1}');
          if [[ $pod_checksum != $local_checksum ]]; then
                 echo "checksum for $i is not equal $pod_checksum $local_checksum"
          fi
   done
   ```
   If the above for-loop did not print anything all the files have been safely
   copied over to the destination
