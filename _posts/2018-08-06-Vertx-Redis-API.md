---
layout: post
title: Vertx Redis API proposal
date: 2018-08-06 12:00:00 GMT
draft: false
---

Once upon the time... [Feb 6, 2013](https://github.com/pmlopes/mod-redis-io/commit/0f889ef12a03b40154fa76a7eaf96f934f8f6a51)... I've started implementing the first [Redis Client](https://redis.io) for [Eclipse Vert.x](https://vertx.io).

The implementation was heavily inspired by [node-redis](http://redis.js.org/) as back in the days I was [working](http://dena.com/intl/) with nodejs on a mobile game title.

The API has slowly evolved during the last 5 years and in the happy scenario it kind of works! However there are some issues, code smells if you prefer...

* The connection is managed by the client not the user
* The client is lazy to connect and caches queries until it is connected
* `AUTH` and `SELECT` are handled by the client
* There are not events for connection errors
* etc...

So at the connection management level there are a couple of issue here, say that the server you are connecting to, is not available, in this case your query request will start queueing in the client, and if this is a critical service (e.g.: cache lookup) you can quickly `OutOfMemory` yourself.

Also if you are using `Sentinel` you can connect to the sentinel node but if you need `AUTH` for the server, you will have to provide it too to the `Sentinel` as the connection handling is magic.

On failover, the client won't follow the connection to a slave but keep retrying the dead server until it comes up if it comes up...

Enough of connections... let's walk over the API...

The [API](https://github.com/vert-x3/vertx-redis-client/blob/master/src/main/java/io/vertx/redis/RedisClient.java) is currently a `> 3000 loc` file with several implementations:

* Client
* Sentinel

And to make things more interesting is manual work. This means that every time there is a new redis release, a human, need to go over the API and see what are the changes, what is added, was anything removed? and adapt these files... The obvious issues are that this is very error prone and will cause many issues for early adopters as the client is not released very often...

## Let's try to fix this!

My proposal is to fix this connection and API issues, in order to make the client easier to maintain is to make it non magical. For example the API would look like this:

```java
Redis redis = Redis.create(vertx, inetSocketAddress(6379, "localhost"));
redis.exceptionHandler(System.err::println);

redis.open(open -> {
  // we are now connected, so we can use redis
});
```

Or a more elaborated example:

```java
Redis redis = Redis.create(vertx, inetSocketAddress(6379, "localhost"));
redis.exceptionHandler(System.err::println);

redis.open(open -> {
  redis.send("ping", send -> {
    if (send.succeeded()) {
      // we should have received a PONG!
    }
  });
});
```

So what is interesting here is that now you can control what will happen if there is an error, and only start sending commands once there is an `open` connection.

Another improvement is support for `batching`, although redis supports [transactions](https://redis.io/topics/transactions), there are times you might want to send a bunch of commands at once without interleaving messages (as it can happen in the current API):

```java
redis.batch(Arrays.asList(
    new Command().setCommand("MULTI"),
    new Command().setCommand("SET").setArgs(Args.args().add("a").add(3)),
    new Command().setCommand("LPOP").setArgs(Args.args().add("a")),
    new Command().setCommand("EXEC")), batch -> {
      // The 4 commands will be sent without other
      // requests interleaving in a sequence
});
```

What you will see is that the API moved from the high level form to a low level, in order to make users having a easier migration path, by having a simple `Handlebars` [template](https://github.com/vert-x3/vertx-redis-client/blob/issues/68/tools/commands.hbs) that generates a simple Java interface.

The benefits here are that at every build/release we can just rerun the template and get the API updated.

## Sentinel

Now that the API is simple, we can build other features really quick, for example Sentinel support is available as:

```java
Sentinel sentinel = Sentinel.create(
  vertx,
  Arrays.asList(
    SocketAddress.inetSocketAddress(26739, host),
    SocketAddress.inetSocketAddress(26740, host),
    SocketAddress.inetSocketAddress(26741, host)
  ));

  // get a connection to the master node
  sentinel.open(Role.MASTER, open -> {
    // query the info
    open.result().send("INFO", info -> {
      // ...
    });
});
```

The Sentinel code will use the given sentinels servers to locate a node with the desired role:

* SENTINEL
* MASTER
* SLAVE

It is also possible to specify the master name in the case there are several masters

## Pub/Sub

Pub sub is also available and does not require an `EventBus` address or magic from the client to spawn multiple connections:

```java
// create the subscriber
sub = Redis.create(vertx, inetSocketAddress(6379, "localhost"));
pub.exceptionHandler(System.err::println);

sub.open(subOpen -> {
  // subscriber is connected, so subscribe to the
  // weather report
  sub.send("SUBSCRIBE", "weather")
});

sub.handler(message -> {
  // just received a weather report
});

// create the publisher
pub = Redis.create(vertx, inetSocketAddress(6379, "localhost"));
pub.exceptionHandler(System.err::println);

pub.open(pubOpen -> {
  // we're open so, lets publish the weather report
  pub.send("PUBLISH", "Sun with chance of Rain");
});
```

## Next

Next step will be implement the redis [cluster](https://redis.io/topics/cluster-spec) spec if there is some demand for it... Thanks for reading and happy Redis!
