---
layout: post
title: Polyglot Proxy Services
date: 2017-01-25 13:00:00 GMT
---

Vert.x offers developers a simple way to define service interfaces with its [Service Proxy](http://vertx.io/docs/vertx-service-proxy/java/) module. The idea is that a developer defines a Java interface and that is the contract between the services across the Vert.x cluster.

Until now this protocol was only implemented in Java so all the polyglot users would not benefit from it, but was this a limitation?

No, just because the contract is a Java interface it does not mean that it cannot be implemented by any other language. For example, say that you have the contract:

```java
@VertxGen
@ProxyGen
public interface MyService {

  void sayHello(Handler<AsyncResult<String>> handler);

  static MyService createProxy(Vertx vertx, String address) {
    return ProxyHelper.createProxy(MyService.class, vertx, address);
  }

  static void registerService(Vertx vertx, String address, MyService service) {
    ProxyHelper.registerService(MyService.class, vertx, service, address);
  }
}
```

One can easily implement this in `JavaScript` and expose on the cluster as:

```js
var Future = require('vertx-js/future');
var MyService = require('jetdrone-services-js/my_service');
// force nashorn to convert to this type later on
var JMyService = Java.type('io.jetdrone.services.MyService');

MyService.registerService(
  vertx,
  'io.jetdrone.services',
  // this is the JS facade
  new MyService(
    // this is the Java Interface implementation
    new JMyService({
      sayHello: function (handler) {
        handler.handle(Future.succeededFuture('Hello there!'));
      }
    })
  )
);
```

For the full example, see [here](https://github.com/pmlopes/vertx3-nashorn.next/tree/master/services).
