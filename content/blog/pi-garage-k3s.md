+++ 
date = 2020-07-05
title = "Raspberry Pi garage opener on k3s cluster"
description = "Raspberry Pi Garage door opener on kubernetes cluster using nodejs"
slug = "" 
tags = ["raspberry-pi","garage-door","kubernetes","docker", "nodejs"]
categories = []
externalLink = ""
series = []
socialShare=true
+++

There are many articles out there which demonstartes how to use a raspberry pi as a DIY garage door opener project. Few are outdated and not deployed using containers images. I found couple good solutions on google but i wasn't able to run them on kubernetes cluster either due to older packages or no enough information. I decided to build my own solution from different sources of information i found

What we'll cover in this post

- setup nodejs project to simulate a button
- create a container from nodejs application
- deploy container on kubernetes cluster

Kudos! to author for this awesome [article](https://gyandeeps.com/garage-operations-raspberrypi/) on showing us how to connect relay and magentic switch to raspberry pi for our purpose.

Once you are done wiring up all components. Let's start setting up our Nodejs application

### Application setup

Create a npm project

```bash
mkdir garage-pi && cd garage-pi
npm init -y
```

We'll be using [node-rpio](https://github.com/jperkin/node-rpio) package which provides access to the Raspberry Pi GPIO interface

Install `node-rpio` and `express` packages

```bash
npm i rpio express -S
```

Create an express app which starts on port `8080` in `server.js` file

```javascript
"use strict";

const express = require("express");
const rpio = require("rpio");

const app = express();
const PORT = 8080;
app.use("/assets", express.static("assets"));

app.listen(PORT);
console.log("Running on http://localhost:" + PORT);
```

Below code let's you simulate a button press; In our case pin number is `19`. First, we want to output to low and set the pin to high after 1000ms. Please refer to `rpio` [repo](https://github.com/jperkin/node-rpio) for more explaination

```javascript
const openPin = process.env.OPEN_PIN || 19;
const relayPin = process.env.RELAY_PIN || 11;

app.get("/relay", function (req, res) {
  // Simulate a button press
  rpio.write(relayPin, rpio.LOW);
  setTimeout(function () {
    rpio.write(relayPin, rpio.HIGH);
    res.send("done");
  }, 1000);
});
```

To get the state of pin

```javascript
function getState() {
  return {
    open: !rpio.read(openPin),
  };
}

app.get("/status", function (req, res) {
  res.send(JSON.stringify(getState()));
});
```

Complete `server.js` file

```javascript
"use strict";

const express = require("express");
const rpio = require("rpio");

const app = express();
const PORT = 8080;
const openPin = process.env.OPEN_PIN || 19;
const relayPin = process.env.RELAY_PIN || 11;

app.use("/assets", express.static("assets"));

function getState() {
  return {
    open: !rpio.read(openPin),
  };
}

app.get("/status", function (req, res) {
  res.send(JSON.stringify(getState()));
});

app.get("/relay", function (req, res) {
  // Simulate a button press
  rpio.write(relayPin, rpio.LOW);
  setTimeout(function () {
    rpio.write(relayPin, rpio.HIGH);
    res.send("done");
  }, 1000);
});

app.listen(PORT);
console.log("Running on http://localhost:" + PORT);
```

### Container image

- Create a dockerfile with [multi-stage](https://docs.docker.com/develop/develop-images/multistage-build/) builds using [nodejs arm image](https://hub.docker.com/r/arm32v7/node/)
- Install necessary python package for `rpio`
- Build docker image
- Publish docker image to your repo in [docker hub](https://hub.docker.com/)
- Create new kubernetes deployment and service

```Dockerfile
# Fetch node_modules for backend, nothing here except
# the node_modules dir ends up in the final image
FROM arm32v7/node:12.18-alpine as builder
RUN mkdir /app
WORKDIR /app
ENV PATH /app/node_modules/.bin:$PATH
COPY package.json /app/package.json
RUN apk add --no-cache make gcc g++ python && \
    npm install --production --silent && \
    apk del make gcc g++ python
RUN npm install

# Add the files to arm image
FROM arm32v7/node:12.18-alpine
RUN mkdir /app
WORKDIR /app
ENV PATH /app/node_modules/.bin:$PATH

# Same as earlier, be specific or copy everything
ADD package.json /app/package.json
ADD package-lock.json /app/package-lock.json
ADD . /app

COPY --from=builder /app/node_modules /app/node_modules

ENV PORT=8080
EXPOSE 8080
CMD [ "npm", "start" ]
```

Docker [buildx](https://docs.docker.com/buildx/working-with-buildx/) feature lets you build arm based images on mac or windows system

```bash
docker buildx build --platform linux/arm64 -t <docker-username>/garage-pi .
docker push <docker-username>/garage-pi
```

Create a new deployment named `garage-pi` that runs earlier published image.

```bash
kind: Deployment
apiVersion: apps/v1
metadata:
  name: garage-pi
  labels:
    app.kubernetes.io/name: garage-pi
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/instance: garage-pi
      app.kubernetes.io/name: garage-pi
  template:
    metadata:
      creationTimestamp: null
      labels:
        app.kubernetes.io/instance: garage-pi
        app.kubernetes.io/name: garage-pi
    spec:
      volumes:
        - name: dev-snd
          hostPath:
            path: /dev/mem
            type: ''
      containers:
        - name: garage-pi
          image: '<docker-username>/garage-pi:latest' ##update username here
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          resources: {}
          volumeMounts:
            - name: dev-snd
              mountPath: /dev/mem
          livenessProbe:
            httpGet:
              path: /
              port: http
              scheme: HTTP
          readinessProbe:
            httpGet:
              path: /
              port: http
              scheme: HTTP
          imagePullPolicy: Always
          securityContext:
            privileged: true
      restartPolicy: Always
```

Create a service for an `garage-pi` deployment, which serves on port `8080` and connects to the containers on port `8080`.

```bash
kubectl expose deployment nginx --port=8080 --target-port=8080
```

That's it, you should be able to hit endpoint and simulate button click!

### Frontend

For frontend of application we'll use [pugjs](https://pugjs.org/api/getting-started.html) templating engine

Install `pug` package

```bash
npm i pug -S
```

1. add `views` folder in root directory and create a new `index.pug` file in views folder
2. wire up templating engine with application
3. render index page on root `/` endpoint
4. integrate button with `/relay` endpoint, which will open/closed garage door

```pug
doctype html
head
  meta(charset='utf-8')
  meta(name='viewport', content='width=device-width, initial-scale=1, shrink-to-fit=no')
  meta(name='description', content='')
  meta(name='author', content='')
  title Garage Opener
.text-center
form.form-signin(method='POST', action='/relay')
  h1.h1.mb-2.font-weight-normal(style='color: #FFFFFF') Garage Door
  .text-center
    .form-signin
      .text-center
      #open
        h1.h2.mb-3.font-weight-normal(style='color: #CF6679')  The Door is Open &#xF62B;
  button#mainButton.btn.btn-lg.btn-primary.btn-block(type='submit') Close
```

```javascript
app.set("views", path.join(__dirname, "views"));
app.set("view engine", "pug");

app.get("/", function (req, res) {
  res.render("index", getState());
});
```

### Twilio Integration (Optional)

Install `dotenv`, `twilio` and `node-schedule` packages

```bash
npm i dotenv node-schedule twilio -S
```

Create a `.env` file at root of the project and add your twilio auth key, account sid and phone number

```env
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=
TWILIO_PHONE_NUMBER=
```

Create a `twilio.js` file, add below code

```javascript
require("dotenv").config();

const accountSid = process.env.TWILIO_ACCOUNT_SID;
const authToken = process.env.TWILIO_AUTH_TOKEN;

const sendSms = (phone, message) => {
  const client = require("twilio")(accountSid, authToken);
  client.messages
    .create({
      body: message,
      from: process.env.TWILIO_PHONE_NUMBER,
      to: phone,
    })
    .then((message) => console.log(message.sid));
};

module.exports = sendSms;
```

Schedule a job for every 15mins to check state of garage door, if door is open we'll send a message

```javascript
const sendSms = require("./twilio");
...
...
schedule.scheduleJob("*/15 * * * *", function () {
  var status = JSON.parse(JSON.stringify(getState()));
  if (status.open) {
    sendSms("<YOUR-NUMBER>", "Garage door is open 🔥");
  }
});
```
