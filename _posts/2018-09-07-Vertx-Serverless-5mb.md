---
layout: post
title: Vert.x + Graal = Java for Serverless = ❤️!
date: 2018-09-07 12:00:00 GMT
draft: false
---

As a follow up after the vacations on the post [Vert.x native image awesomeness!](https://www.jetdrone.xyz/2018/08/10/Vertx-native-image-10mb.html),
I got some Twitter challenge about what about serverles?

![Twitter](/assets/images/blog/twitter-201809.png)

This led to a couple of discussions and catching back with some interest on the [OpenFaaS](https://www.openfaas.com) project. From here to create a
simple [template](https://github.com/pmlopes/openfaas-svm-vertx) was just a couple of minutes.

So what is interesting about this? Well first the fact that we can use `Java` as a viable platform to write `Functions`. Functions are by definition, small units of code and easy to write. Given that [Vert.x](https://vertx.io) is a light framework making a function was a piece of cake!

## OpenFaaS

From the [OpenFaaS site](https://www.openfaas.com/) you can read:

> Run functions anywhere with the same unified experience - bring your laptop, your own on-prem hardware or create a cluster in the cloud. Pick Kubernetes or Docker to do the heavy lifting enabling you to build a scalable, fault-tolerant event-driven serverless platform for your applications.

## The template

In order to run the example and template all you need is to setup your environment and install the `faas-cli` tool. For this I'd recommend you to
read the nice docs on OpenFaaS.

Once the setup is complete we can start playing with Vert.x and Graal as a Function:

### Get the template

The first step is to get the template, for this you need to run:

```
$ faas-cli template pull https://github.com/pmlopes/openfaas-svm-vertx

Fetch templates from repository: https://github.com/pmlopes/openfaas-svm-vertx
2018/09/07 11:59:02 Attempting to expand templates from https://github.com/pmlopes/openfaas-svm-vertx
2018/09/07 11:59:05 Fetched 1 template(s) : [vertx-svm] from https://github.com/pmlopes/openfaas-svm-vertx
```

You should see some output similar to the listing above. Now that you have the template locally we can create our first function:

```
$ faas-cli new callme --lang vertx-svm

Folder: callme created.
  ___                   _____           ____
 / _ \ _ __   ___ _ __ |  ___|_ _  __ _/ ___|
| | | | '_ \ / _ \ '_ \| |_ / _` |/ _` \___ \
| |_| | |_) |  __/ | | |  _| (_| | (_| |___) |
 \___/| .__/ \___|_| |_|_|  \__,_|\__,_|____/
      |_|


Function created in folder: callme
Stack file written: callme.yml
```

As this moment `OpenFaaS` just created the skeleton project for you, you can inspect it:

```
tree callme
callme
├── pom.xml
├── README.md
└── src
    └── main
        ├── java
        │   └── MyFunction.java
        ├── resources
        │   └── META-INF
        │       └── services
        │           └── xyz.jetdrone.openfaas.vertx.OpenFaaS
        └── svm
            └── reflection.json

7 directories, 5 files
```

As you can see it's a trivial maven project with a single source file `MyFunction.java`. A close look at the source file you will find the code:

```java
import io.vertx.ext.web.RoutingContext;

import xyz.jetdrone.openfaas.vertx.OpenFaaS;

public class MyFunction implements OpenFaaS {

  public MyFunction() {
    System.out.println("Loaded MyFunction!");
  }

  @Override
  public void handle(RoutingContext ctx) {
    ctx.response().end("Hi!");
  }

}
```

The constructor is optional so we could make the function even more minimal with:

```java
import io.vertx.ext.web.RoutingContext;

import xyz.jetdrone.openfaas.vertx.OpenFaaS;

public class MyFunction implements OpenFaaS {
  @Override
  public void handle(RoutingContext ctx) {
    ctx.response().end("Hi!");
  }
}
```

If you were paying attention, there isn't anything new here, the function is in fact a [vertx-web](https://vertx.io/docs/vertx-web/java/) handler,
so you don't need to learn a new framework to get into serverless. But one might ask why the need for a `OpenFaaS` interface then? The reason is
because the entry point application will use [Service Loaders](https://docs.oracle.com/javase/8/docs/api/java/util/ServiceLoader.html) to load any
available implementations of the marker interface and will add them all to the server.

## Build the function

```
$ faas-cli build -f callme.yml

...

Successfully built f5e1266b0d32
Successfully tagged callme:latest
Image: callme built.
```

The template will use a Dockerfile that will perform all the heavy work, grab a coffee until you'll see the message above.

## Deploy the function

```
$ faas-cli deploy -f callme.yml

Deploying: callme.
No existing service to remove
Deployed.
200 OK
URL: http://localhost:8080/function/callme
```

You can verify it using the web frontend usually running at `http://localhost:8080` and it should look more or less like this:

![OpenFaaS-Portal](/assets/images/blog/OpenFaaS-201809.png)

## Play with it

Either `cURL`, `httpie` or use your browser at `http://localhost:8080/function/callme`.

```
http http://127.0.0.1:8080/function/callme
HTTP/1.1 200 OK
Content-Length: 3
Content-Type: text/plain; charset=utf-8
Date: Fri, 07 Sep 2018 12:54:03 GMT
X-Call-Id: a2978cfb-33de-48aa-bc1e-75d370834e87
X-Start-Time: 1536324843677727462

Hi!
```

Have fun!

## Docker stats

So when we start our application and postgres and run `docker stats` we can see:

```
CONTAINER ID        NAME                                            CPU %               MEM USAGE / LIMIT
6889f23e814b        func_queue-worker.1.pa37e6ysx20dhfjpcdv9gpite   0.00%               2.496MiB / 50MiB
f9025c9ce3ca        func_gateway.1.qyvg40lzd60o5xez65y3t8rg3        1.34%               8.898MiB / 15.55GiB
a175047e7fe5        callme.1.jbqbp8sd4psz6io1t0uhiq64m              2.49%               7.105MiB / 15.55GiB
40110b24cf82        func_nats.1.7gnn1jpmxe2kepqahrjjdeati           0.06%               5.875MiB / 125MiB
cb00f7c89f32        func_alertmanager.1.kcwlcpm1kmq7gpqvy0t91psxa   0.14%               6.922MiB / 50MiB
ed196726afbd        func_prometheus.1.7vbsnyrytyl9hqu95w8z3ju3d     1.73%               33.34MiB / 500MiB
07dc59c14cf0        func_faas-swarm.1.yeh0xm62rhkebwlhmueqpv47f     0.00%               7.715MiB / 15.55GiB
^C
```

I'd say that it **amazing** a complex application running in **7MB** of RAM which accounts **BOTH** for the function **AND** the watchdog!!!
