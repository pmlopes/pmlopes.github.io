---
layout: post
title: "Trust me! SSL works on native images"
date: 2019-04-16 08:00:00 GMT
draft: false
---

Native Images and [SubstrateVM](https://github.com/oracle/graal/tree/master/substratevm), the native image AOT compiler from GraalVM, are the hot topic, almost everyone has build a simple HTTP server. But not many people talk about SSL.

Building an `HTTPS` server is not difficult. For example this is how you would create a client in [Eclipse Vert.x](https://vertx.io):

```java
WebClient client = WebClient
    .create(
      Vertx.vertx(),
      new WebClientOptions().setSsl(true));

client
  .get(443, "icanhazdadjoke.com", "/")
  .putHeader("Accept", "text/plain")
  .send(ar -> {
    if (ar.succeeded()) {
      System.out.println(ar.result().body());
    }
});
```

Using SSL is nothing new, in fact I already demoed it at [JFall 2018](https://www.youtube.com/watch?v=4Ok7t9oXCzw).

<amp-youtube data-videoid="4Ok7t9oXCzw" layout="responsive" width="480" height="270"></amp-youtube>


However in that demo I was just trusting all certificates. Trust is something we usually don't do on the web so I'll
now show how you can make the code above run by fully verifying the certificate associated with the site I'm consuming
the jokes API.

### Step 1

First step we need to enable security services, for this we need to use the extra flag to `native-image`:

```
--enable-all-security-services
```

### Step 2

Enabling security services will require that the dll `libsunec` is available in your path. From a linux distribution of `GraalVM` copy the file:

```
cp $GRAALVM_HOME/jre/lib/amd64/libsunec.so libsunec.so
```

This will allow you to use SSL/Security functions. But the application will still not run:


```
Caused by: io.netty.handler.codec.DecoderException:
    java.lang.RuntimeException: Unexpected error:
        java.security.InvalidAlgorithmParameterException:
            the trustAnchors parameter must be non-empty
...
```

### Step 3

All functions are in place but the application will fail to run because the certificate cannot be verified, now it seems that the final native images
will not look to the system root certificates (the `ca-certificates` package in many distributions) or sometimes the format is not recognized (should be
`PCKS12`). So we need to copy that too from `GraalVM`:

```
cp $GRAALVM_HOME/jre/lib/security/cacerts cacerts
```

### Step 4

Yet this won't work because the files will not be read from the expected location, to solve this we can tell the native image where to read the ca certificate:

```
./dad-jokes \
  -Djavax.net.ssl.trustStore=./cacerts \
  -Djavax.net.ssl.trustAnchors=./cacerts


A doll was recently found dead in a rice paddy.
It's the only known instance of a nick nack paddy wack.
```

And *voilá* it works!
