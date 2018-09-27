---
layout: post
title: Who needs coroutines in Java when there's ES4X?
date: 2018-09-27 12:00:00 GMT
draft: false
---

It has been an amazing week for [ES4X](https://reactiverse.io/es4x/) with the first results of the [Techempower Benchmark](https://www.techempower.com/benchmarks/#section=test&runid=586ebf0d-4369-4648-8293-590ab295e64d&hw=ph&test=db) showing the polyglot aspect of [Eclipse Vert.x](https://vertx.io) as the **4th** fastest framework of them all, and the **FASTEST** using `JavaScript`.

One of the things `ES4X` offers is the support for modern JavaScript on the JVM (thanks to [GraalVM](http://www.graalvm.org/)). So after reading and chatted about the future of Java and the inclusion of *co-routines* into the JVM with project [Loom](http://openjdk.java.net/projects/loom/) got me to think:

> Why wait for Loom when there's JavaScript?
> ¯\\_(ツ)_/¯

Let me explain this a bit further and do a small exercise with the Techempower code. Lets look at the code that fetches a single line of the postgres database and returns it to the client, the initial code is like this:

```js
import {Router} from '@vertx/web';
import {PgClient, Tuple} from '@reactiverse/reactive-pg-client';
import {PgPoolOptions} from '@reactiverse/reactive-pg-client/options';

const SELECT_WORLD = "SELECT id, randomnumber from WORLD where id=$1";

const app = Router.router(vertx);

let client = PgClient.pool(
  vertx,
  new PgPoolOptions()
    .setUser('benchmarkdbuser')
    .setPassword('benchmarkdbpass')
    .setDatabase('hello_world'));

app.get('/').handler(
  ctx => {
    client.preparedQuery(SELECT_WORLD, Tuple.of(1), res => {
      if (res.succeeded()) {
        let resultSet = res.result().iterator();

        if (!resultSet.hasNext()) {
          // no result found
          ctx.fail(404);
          return;
        } else {
          let row = resultSet.next();

          ctx.response()
            .putHeader("Content-Type", "application/json")
            .end(JSON.stringify({
              id: row.getInteger(0),
              randomNumber: row.getInteger(1)
            }));
        }
      } else {
        ctx.fail(res.cause());
      }
    });
  }
);

vertx
  .createHttpServer()
  .requestHandler(req => app.accept(req))
  .listen(8080);

console.log('Server listening at: http://localhost:8080/');
```

For the *untrained* java developer eye, there is lots of *indentation* making it non trivial to follow or read. JavaScript developers are used to this kind of code so, if you're a polyglot developer it probably won't hurt your eyes that much to see this...

## I Promise it will be easy!

With `ES6` JavaScript gained a new type [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise). The **Promise** object represents the eventual completion (or failure) of an asynchronous operation, and its resulting value. The spec for `Promise` is quite simple so `ES4X` has it available even when you're using `Nashorn` (Which means only `ES5.1` is supported).

Promises will make your code a bit more readable by chaining calls to: `then()`, `catch()` and `finally()` instead of large callbacks and more indentation.

### Not so fast!!!

Promises can make your code simpler but remember `Vert.x` API's are **callback** based so this won't help much (*This isn't entirely true*). When creating `ES4X` the inspiration came mostly from the **GOOD** practices you see in `nodejs`, so why not having a `system` module named `util` that implements a couple of common features, such as the [promisify](https://nodejs.org/api/util.html#util_util_promisify_original) function?

And that is what you do have already on `ES4X`, but wait, there's more! `ES4X` *promisify* function behaves in two modes:

* when the argument is a function, it behaves just like in nodejs
* when the argument is an object, then it wraps each function on the object

So if we were to focus on the previous handler code we could re-write the handler as:

```js
const util = require('util');

app.get('/').handler(
  ctx => {
    // promisified client, methods will return a promise
    // and will not take a callback as the last argument
    let postgres = util.promisify(client);

    postgres
      .preparedQuery(SELECT_WORLD, Tuple.of(1))
      .then(rs => {
        let resultSet = rs.iterator();

        if (!resultSet.hasNext()) {
          ctx.fail(404);
        } else {
          let row = resultSet.next();
          ctx.response()
            .putHeader("Content-Type", "application/json")
            .end(JSON.stringify({
              id: row.getInteger(0),
              randomNumber: row.getInteger(1)
            }));
        }
      })
      .catch(e => ctx.fail(e));
  }
);
```

This is a bit of a improvement, we traded a small wrapper by **code readability**.

## ES6 is so 2017...

*JavaScript* world evolves really fast and although *ES6* is great, developers have already moved on to **ES7**! Thankfully `GraalVM` supports many features of `ES7`, and the feature I'm really interested now is `async-await`. The premise is simple:

> Within `async` function(s) you can `await` for results!

And you can await on... `Promises`!!! so this means that we can make our code even simpler by using a `async` function and replace our promise chain with `await` like this:

```js
const util = require('util');

app.get('/').handler(
  ctx => {
    (async function () {
      // vert.x promise
      let postgres = util.promisify(client);

      try {
        let rs = await postgres.preparedQuery(SELECT_WORLD, Tuple.of(1));
        let resultSet = rs.iterator();

        if (!resultSet.hasNext()) {
          ctx.fail(404);
        } else {
          let row = resultSet.next();
          ctx.response()
            .putHeader("Content-Type", "application/json")
            .end(JSON.stringify({
              id: row.getInteger(0),
              randomNumber: row.getInteger(1)
            }));
        }
      } catch (e) {
        ctx.fail(e);
      }
    })();
  }
);
```

As you can now read, we've flattened out code using the `await` keyword so the `try`, `catch` block is now readable... but there's a **🤢** in this code, in order to use the `await` language feature, we must call all our code from an `async` function, so the whole handler is now wrapped in a `async IIFE`.

### Do we really need it?

The question now is, how can we make things more readable again? and the answer is in front of our eyes. If you pay close attention, the `handler` function of the router, takes as an argument a closure. A closure in *JavaScript* is a simple function, so why not make this *closure* an `async` one?

```js
const util = require('util');

app.get('/').handler(
  async ctx => {
    let postgres = util.promisify(client);

    try {
      let rs = await postgres.preparedQuery(SELECT_WORLD, Tuple.of(1));
      let resultSet = rs.iterator();

      if (!resultSet.hasNext()) {
        ctx.fail(404);
      } else {
        let row = resultSet.next();
        ctx.response()
          .putHeader("Content-Type", "application/json")
          .end(JSON.stringify({
            id: row.getInteger(0),
            randomNumber: row.getInteger(1)
          }));
      }
    } catch (e) {
      ctx.fail(e);
    }
  }
);
```

Easy, just add `async` before the `ctx` variable and get rid of the `IIFE`!

# Conclusion

With `ES4X` and `GraalVM` you can already use all the modern features of *JavaScript* out of the box, no need to transpilers or compilation steps, on top of that you will have the **FASTEST** runtime for your JavaScript code and the possibility of using the best of both Worlds (NPM modules and Maven modules), so what are you waiting for to try it out? You can check out the source code for this article [here](https://gist.github.com/pmlopes/ce9becabcb4f147e0df3168a596acaa3). And of course running it is a simple as:

```
npm i
npm start
```

What are you waiting for? :-)
