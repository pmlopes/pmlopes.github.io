---
layout: post
title: Vert.x native image awesomeness!
date: 2018-08-10 12:00:00 GMT
draft: false
---

I've been working with [GraalVM](http://graalvm.org) for a while now mostly on the [Polyglot](https://reactiverse.io/es4x/) aspect, but some work has been done in the [native image](https://github.com/eclipse/vert.x/pull/2526) support. Today I'd like to share what it is possible right now, *spoiler*: run a Docker image with **size** of **38MB** and **10MB** of **RAM**!

## A bit of history

With the 3.x series of Eclipse Vert.x lots of work was done to reduce the ammount of *Classloaders*, *Reflection magic* (this is why there are no annotations, for example). This usually is seen as a contra sense in the java world, but it paid off, as we can already do some amazing stuff without having to thinkering much on hacks or workarounds to circunvent the [substrate limitations](https://github.com/oracle/graal/blob/master/substratevm/LIMITATIONS.md).

## Proof of Concept

As a proof of concept, I've decided to build a small application based on a presentation I've made about 2 years ago at [JavaZone](https://github.com/pmlopes/javazone-2016). The application does the following:

* **Start a WebSocket client** to collect in **realtime** the unconfirmed bitcoin transactions.
* **Connect to Postgres** with the winner of [Techempower Benchmarks #16](https://www.techempower.com/benchmarks/#section=data-r16&hw=ph&test=db) [pg-reactive-client](https://reactiverse.io/reactive-pg-client/).
* On *unconfirmed transaction* save it to postgres.
* On *save* **publish** the *UTX* map-reduced value to **vert.x eventbus**
* Start a **HTTP server** serve a *single page application*
* On *server start* **bridge the eventbus** from the server to the webbrowser using **sockjs**
* On the browser, render the *UTX* values in *realtime* as they arrive from the **server websocket**.

As you can see there is already a lot happening here, **servers**, **clients**, **websockets**, **eventbus**. I could keep adding more stuff but it's a *PoC* right!

## Show me the code

Now that we have the idea, we need to implement it, as I said, this was a remake of an old presentation so the code was almost all done for me. This is how it turned out to be:

```java
final Vertx vertx = Vertx.vertx();
final EventBus eb = vertx.eventBus();
final PgPool postgres = PgClient.pool(...);

BlockChainClient.create(vertx)
  .connect(self -> {
    self.subscribeUnconfirmed(json -> {
      postgres.preparedQuery(
        "INSERT INTO UTX (data) VALUES ($1)",
        Tuple.of(Json.create(json)), ar -> {
          int utxValue = 0;
          if (json.containsKey("out")) {
            // reduce step
            for (Object o : json.getJsonArray("out")) {
              utxValue += ((JsonObject) o).getInteger("value", 0);
            }
          }
          eb.publish("data.updates", utxValue);
      });
    });
  });

final Router app = Router.router(vertx);

app.route("/eventbus/*").handler(SockJSHandler.create(vertx));
app.route().handler(StaticHandler.create("static"));

vertx.createHttpServer()
  .requestHandler(app)
  .listen(Integer.getInteger("port", 8080));
```

*Error handling and configuration ommited for brevity*.

This is a very simple application and in fact this is **exactly** how you would code it with Vert.x, I did **not** do any hack to get it to run on Graal SVM.

## Build it

Building the code is trivial, but building the image turned out to have some issues. Luckyly the issues are simple and can all be addressed with a simple [substitution java file](https://github.com/pmlopes/vertx-starter/blob/master/templates/graal-nativeimage/src/main/svm/substitutions.java).

So building the native image was as simple as:

```
native-image \
  -Djava.net.preferIPv4Stack=true \
  -Dvertx.disableDnsResolver=true \
  -H:IncludeResources="(META-INF/services|webroot)/.*" \
  -H:+ReportUnsupportedElementsAtRuntime \
  -H:ReflectionConfigurationFiles=./src/main/svm/reflection.json \
  -jar "target/bitcoin-viewer.jar"
```

So what is important to see here, first I specify that I prefer the **IPV4** stack from the VM (but this is not really a requirement). Second, I disable **Vert.x DNS resolver** for the reason that it is not working correctly with SVM and vert.x will fallback to the VM default (blocking) DNS resolver which works fine. I then specify what resources I'd like to keep from my fatjar and specify the substitution file.

Once it completes you can see:

```
-rwxrwxr-x.  1 plopes plopes 24603024 10 aug 11:45 bitcoin-viewer
```

Your application is now just **24,6MB**!!!

## Dockerize it

Graal SVM relies on **Glibc** so this means you can run it on many images such as *ubuntu*, *fedora*, *centos*... but these images are quite big, so why not *alpine*?

Alpine images are build against *musl-libc* and this is an issue. Luckily the alpine project has documented how to get *glibc* working on alpine and there are already some images on docker hub like `frolvlad/alpine-glibc`.

So lets convert to docker shall we?

```Dockerfile
# GraalVM docker image used for AoT compilation
FROM panga/graalvm-ce:latest AS build-aot
# Add maven wrapper
ADD mvnw app/
ADD mvnw.cmd app/
ADD .mvn app/.mvn/
# Add settings.xml to allow snapshots
ADD settings.xml root/.m2/
# Add pom
ADD pom.xml app/
# Add sources
ADD src app/src/
# Set working dir
WORKDIR /app
# Build (java side)
RUN ./mvnw -Pnative-image package
# Build image
RUN native-image \
    --no-server \
    -Djava.net.preferIPv4Stack=true \
    -Dvertx.disableDnsResolver=true \
    -H:IncludeResources="(META-INF/services|webroot)/.*" \
    -H:+ReportUnsupportedElementsAtRuntime \
    -H:ReflectionConfigurationFiles=./src/main/svm/reflection.json \
    -jar "target/bitcoin-viewer.jar"
# Create new image from alpine
FROM frolvlad/alpine-glibc:alpine-3.8
RUN apk add --no-cache ca-certificates
# Copy generated native executable from build-aot
COPY --from=build-aot /app/bitcoin-viewer /bitcoin-viewer
# Set the entrypoint
ENTRYPOINT [ "/bitcoin-viewer" ]
```

## Results

So when we start our application and postgres and run `docker stats` we can see:

```
CONTAINER ID  NAME            CPU %  MEM USAGE / LIMIT    MEM %
c7f5e7af56fe  vertx-bitcoin   0.61%  5.133MiB / 15.54GiB  0.03%
a3536684f175  postgres        1.26%  11.27MiB / 15.54GiB  0.07%
```

I'd say that it **amazing** a complex application running in **5MB** of RAM!!! with instant startup!

And about the image sizes, running `docker images` we see:

```
REPOSITORY      TAG      IMAGE ID       CREATED             SIZE
vertx-bitcoin   latest   3b8b487b8ad8   43 minutes ago      37.9MB
```

So the total size is **38MB**!!!

Java is now a viable **Serveless** technology and a it we can agree that java ain't slow and bloated right?!
