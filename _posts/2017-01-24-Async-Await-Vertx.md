---
layout: post
title: Async Await (Bye bye callback hell) with Vert.x!
date: 2017-01-24 13:00:00 GMT
---

One of the common complains while developing with event loop platforms (Vert.x, Node.js, etc...) is that you will sonner or latter face callback hell.

Callback hell is a syntom of poorly organized code or the drawbacks of callback style programming where returns are not blocked until a response arrives from the callee.

Vert.x provided some tools to help fighting this issue:

* `executeBlocking` (not really a solution since it just delegates to an good old thread pool)
* RXified API's
* Completable Future API's
* `Vert.x-Sync` for the Java developers

Now if you're doing `JavaScript` development then you can add another tool to the toolbelt. Armed with `Webpack` and `Babel` one can `promisify` the Vert.x API and use promisses. What is so amazing with this is that there are babel plugins that can now transforms `async` `await` calls to Promisses and this will then allow us to escape the callback hell.

Here is an example:

```js
// get a reference to the web webclient class from java
const WebClient = Java.type('io.vertx.webclient.WebClient');

// in order to use await one needs to make a call from a async function, so we wrap our main code into
// and async function
(async function () {
  // error handling can be done with try catch
  try {
    let webClient = WebClient.create(vertx).get(80, "www.google.com", "/");
    // since vertx API is callback style we need to convert that into a Promise style, either you do it manually
    // or use this helper that wraps a object and returns a proxy that adds a extra last parameter to all method
    // calls that handles the async result objects
    let response = await Promise.devertxify(webClient).send();
    // there was no threads involved in this code, the call was async and run on vert.x event loop
    // however you did not need to create callbacks and chain functions! hurray!!!
    console.log("Received response with status code: " + response.statusCode());
  } catch (e) {
    // oops! there was an error on the callback but now it is all managed as if it was blocking code,
    // just handle the exception in a try catch block!
    console.error(e);
  }
})();
```

## How it works

All you need is enable the transformation in `.babelrc`:

```js
{
  "presets": ["es2015"],
  "plugins": ["async-to-promises"]
}
```

A link to the full example is [here](https://github.com/pmlopes/vertx3-nashorn.next/tree/master/examples/async-await).
