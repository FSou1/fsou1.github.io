---
title: NodeJS general guide
tags: JavaScript NodeJS
redirect_from: "/NodeJS_guide/"
---

This post is a collection of notes related to general overview and guides of Nodejs.

# How to start a web server

Once you've installed Node, create a file named `app.js` with the following code:

```
const http = require('http');

const hostname = '127.0.0.1';
const port = 3000;

const server = http.createServer((req, res) => {
  res.statusCode = 200;
  res.setHeader('Content-Type', 'text/plain');
  res.end('Hello World\n');
});

server.listen(port, hostname, () => {
  console.log(`Server running at http://${hostname}:${port}/`);
});
```

To run the app execude `node app.js`. Once it's done, you can visit `http://localhost:3000` and will see a 'Hello World'.

# How to send/handle HTTP requests

The [http](https://nodejs.org/api/http.html) module allows Node.js to transfer data over HTTP. The module is responsible for:

- Building web servers
- Sending client requests
- Giving responses back to clients
- Managing connections
- Holding HTTP constants (methods, status codes)

To include the HTTP module, use the `require()` method:

```
var http = require('http');
```

The following code will send a `GET` request to NASA’s API and print out the URL for the astronomy picture of the day as well as an explanation:

```
const https = require('https');

https.get('https://api.nasa.gov/planetary/apod?api_key=DEMO_KEY', (resp) => {
  let data = '';

  // A chunk of data has been recieved.
  resp.on('data', (chunk) => {
    data += chunk;
  });

  // The whole response has been received. Print out the result.
  resp.on('end', () => {
    console.log(JSON.parse(data).explanation);
  });

}).on("error", (err) => {
  console.log("Error: " + err.message);
});
```

## Other ways to make HTTP requests

### Request

First install the package `npm install request`. Then a request can be made like this:

```
const request = require('request');

request('https://api.nasa.gov/planetary/apod?api_key=DEMO_KEY', { json: true }, (err, res, body) => {
  if (err) { return console.log(err); }
  console.log(body.url);
  console.log(body.explanation);
});
```

### Axios

Axios is a Promise based HTTP client for the browser as well as node.js.

To install from npm execute `npm install axios`.

And to make a request:

```
const axios = require('axios');

axios.get('https://api.nasa.gov/planetary/apod?api_key=DEMO_KEY')
  .then(response => {
    console.log(response.data.url);
    console.log(response.data.explanation);
  })
  .catch(error => {
    console.log(error);
  });
```

### Got

Got is another Promise based lightweight solution. Again, install as: `npm install got`.

And the code is:

```
const got = require('got');

got('https://api.nasa.gov/planetary/apod?api_key=DEMO_KEY', { json: true }).then(response => {
  console.log(response.body.url);
  console.log(response.body.explanation);
}).catch(error => {
  console.log(error.response.body);
});
```

# How to debug a script

Debugging process can be started in nodejs by adding the `--inspect` flag:

```
node --inspect index.js
```

Once it's done, you will receive the following message:

```
Debugger listening on ws://127.0.0.1:9229/5f063ae5-c5ea-4371-ad7a-325ac9b25493
For help, see: https://nodejs.org/en/docs/inspector
```

After that you can open Chrome DevTools > Dedicated DevTools for Nodejs (green icon) > any other tabs with similar debugging experience. No words, just amazing.

Source: https://nodejs.org/en/docs/guides/debugging-getting-started/

# How to dockerize an existing Nodejs app

First you have to install either Docker Desktop for Windows (Microsoft Windows 10) or Docker toolbox.

Then create a `Dockerfile` with the following content:

```
FROM node:10

# Create app directory
WORKDIR /usr/src/app

# Install app dependencies
# A wildcard is used to ensure both package.json AND package-lock.json are copied
# where available (npm@5+)
COPY package*.json ./

RUN npm install
# If you are building your code for production
# RUN npm ci --only=production

# Bundle app source
COPY . .

EXPOSE 8080
CMD [ "node", "server.js" ]
```

`server.js` should be replaced with the file name you "node".

`.dockerignore` file is useful to prevent your local modules and debug logs from being copied onto a Docker image:

```
node_modules
npm-debug.log
```

Run the docker and build an image using the `docker images` command:

```
docker build -t <your username>/node-web-app .
```

Once the image is built, you can see it in the list:

```
$ docker images

# Example
REPOSITORY                      TAG        ID              CREATED
node                            10         1934b0b038d1    5 days ago
<your username>/node-web-app    latest     d64d3505b0d2    1 minute ago
```

Last but not least, run the image you previously built:

```
docker run -p 49160:8080 -d <your username>/node-web-app
```

The output of your app can be printed with:

```
# Get container ID
$ docker ps

# Print app output
$ docker logs <container id>

# Example
Running on http://localhost:8080
```

Source: https://nodejs.org/en/docs/guides/nodejs-docker-webapp/#creating-a-dockerfile.

More info at:

- Docker
  - [Official Node.js Docker Image](https://hub.docker.com/_/node/)
  - [Node.js Docker Best Practices Guide](https://github.com/nodejs/docker-node/blob/master/docs/BestPractices.md)
  - [Official Docker documentation](https://docs.docker.com/)
