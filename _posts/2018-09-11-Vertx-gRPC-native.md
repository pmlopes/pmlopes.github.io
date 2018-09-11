---
layout: post
title: gRPC Vert.x native image? Yes you can!
date: 2018-09-11 12:00:00 GMT
draft: false
---

During the last couple of weeks I've been identifying and documenting what it takes to get [Eclipse Vert.x](https://vertx.io) applications to work as [GraalVM](http://www.graalvm.org/) native images. Until now tweaks and fixes are in place to allow developers to use many common components (bellow you'll see a small subset of this list), So today I'll be describing [gRPC](https://vertx.io/docs/vertx-grpc/java/).

* [PostgreSQL databases](https://reactiverse.io/reactive-pg-client/)
* [Redis](https://redis.io/)
* [Vert-Web](https://vertx.io/docs/vertx-web/java/)
* [EventBus Bridges](https://vertx.io/docs/vertx-tcp-eventbus-bridge/java/)

## gRPC

The best description of gRPC can be seen at wikipedia:

> gRPC is an open source remote procedure call (RPC) system initially developed at Google. It uses HTTP/2 for transport, Protocol Buffers as the interface description language, and provides features such as authentication, bidirectional streaming and flow control, blocking or nonblocking bindings, and cancellation and timeouts. It generates cross-platform client and server bindings for many languages.

When investigating the code and dependencies of `gRPC` I was concerned that this could be a complex beast to handle, but I was wrong! Actually everything works.

### Bootstrap a gRPC project

In order to simplify your work I've a small static website that generates projects for you, don't worry, it's a static html app, it all runs on your browser, nothing is sent to servers, all your secrets stay with you.

![grpc](/assets/images/blog/grpc.png)

If you get the same options I have you can go ahead and generate your skeleton project. Once you've done this and open the `MainVerticle.java` file you can start coding your own server. However before we get there we need a `proto` file. The proto file will contain the description of your API, since this is a lazy example let's just use the `hello world` example that you can get with `gRPC` and add it to `src/main/proto/helloworld.proto`:

```proto
syntax = "proto3";

option java_multiple_files = true;
option java_package = "io.grpc.examples.helloworld";
option java_outer_classname = "HelloWorldProto";
option objc_class_prefix = "HLW";

package helloworld;

// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}
```

As you can see it's a very simple protocol, there's a `Greeter` service that you can use to send `HelloRequest`s and there's a `HelloReply` that will contain a message back. Knowing this we can write our own server as:

```java
Vertx vertx = Vertx.vertx();

VertxServer server = VertxServerBuilder
  .forAddress(vertx, "127.0.0.1", 50051)
  .addService(new GreeterGrpc.GreeterVertxImplBase() {
    @Override
    public void sayHello(HelloRequest req, Future<HelloReply> fut) {
      System.out.println("Hello " + req.getName());
      fut.complete(
        HelloReply.newBuilder()
          .setMessage("Hi there, " + req.getName())
          .build());
    }
  }).build();

server.start(ar -> {
  if (ar.succeeded()) {
    System.out.println("gRPC service started");
  } else {
    ar.cause().printStackTrace();
    System.exit(1);
  }
});
```

As usual one needs to build it:

```
./mvn -Pnative-image
```

And you will end up with a 18MB binary.

### Writing a client

Of course we'll need a client too, so here's what you need:

```java
ManagedChannel channel = VertxChannelBuilder
  .forAddress(vertx, "localhost", 50051)
  .usePlaintext(true)
  .build();

GreeterGrpc.GreeterVertxStub stub = GreeterGrpc.newVertxStub(channel);

HelloRequest request = HelloRequest.newBuilder().setName("Paulo").build();

stub.sayHello(request, asyncResponse -> {
  if (asyncResponse.succeeded()) {
    System.out.println("Succeeded " + asyncResponse.result().getMessage());
  } else {
    asyncResponse.cause().printStackTrace();
  }
});
```

## Testing

From this moment on, all you need to do is run both application and see the output:

```
server: gRPC service started
server: Hello Paulo
client: Succeeded Hi there, Paulo
```

## Conclusion

I am really impressed with the work the `GraalVM` and `SubstrateVM` team has done, in order to support such complex things as the ones I've got to work:

* vertx
* netty
* postgres
* redis
* gRPC
* protocol buffers
* websockets
* HTTP2
* etc...

It is still in its infancy but `SubstrateVM` is showing the full potential of `AOT` for any kind of *short lived* java application, be it **serverless functions**, **cli** applications, you name it...
