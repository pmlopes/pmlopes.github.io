---
layout: post
title: "Instant Passwordless JVM apps with Vert.x and FIDO2"
date: 2022-01-07 08:00:00 GMT
draft: false
---

The history around passwords is blurry as the invention of the wheel. Most people say that one of the first usages was
in the mid nineteen sixties, when MIT was building the Compatible Time-Sharing System (CTSS). This was a 30 million
dollar computer, that was being shared between users.

        Quickly there was a need to keep private homes for each user. Computing resources and storage were scarce at that time
        so the solution was to store a secret (the password) that would be checked against the user login and verify the
        identity.

Many years later, around 1989, Tim Berners-Lee developed the HTTP protocol which allowed us to have
the internet as we know it today. Since then, many standards have been defined:

* HTTP Basic Authentication - Which encodes usernames and passwords in Base64, which by the way, is not a encryption algorithm.
* HTTP Digest Authentication - Which improves the previous status quo by hashing using MD5 the user credentials with the requested resource HTTP method and URI. By today's standard, MD5 is not considered secure and should be avoided as it can be brute forced in a very short amount of time.
* HTTP Bearer tokens - Which improves the security by using better hashing algorithms, yet doesn't solve issues such as replay attacks
* Form based Authentication - Where developers "invent" their own security, which sadly ends up on big security breaches.
* One Time Password - Using authenticators with a pre shared key. Given that the key is pre-shared, it can be leaked easily.

And we could be keep the list going if we wanted...

### What's wrong with the status quo?

So what is wrong with the status quo? Well, the world has now billions of connected devices to the internet. Dozens of authentication
mechanisms, which sadly all have big flaws. Let me just list a couple of well know security breaches to illustrate
the severity of the problem.

* Yahoo (Aug 2013) 3 billion accounts leaked
* Alibaba (Nov 2019) 1.1 billion user data leaked
* LinkedIn (Jun 2021) 700 million user data leaked
* Sina Weibo (Mar 2020) 538 million accounts leaked
* Facebook (Apr 2019) 533 million user data leaked
* ...

Sadly this is just a scratch of the top of the iceberg. Probably there's a breach happening right now and the list will never end.

In this conference you will learn a lot about FIDO2. I'll leave it's details to other speakers. If you want to know more
I'd recommend you to visit the https://fidoalliance.org/fido2 website.

<amp-youtube data-videoid="E2S-gYdNnuo" layout="responsive" width="480" height="270"></amp-youtube>

### Source Example

```java
package com.example.webauthn;

import io.vertx.core.AbstractVerticle;
import io.vertx.core.Promise;
import io.vertx.core.http.CookieSameSite;
import io.vertx.core.http.HttpServerOptions;
import io.vertx.core.net.JksOptions;
import io.vertx.ext.auth.webauthn.RelyingParty;
import io.vertx.ext.auth.webauthn.WebAuthn;
import io.vertx.ext.auth.webauthn.WebAuthnOptions;
import io.vertx.ext.web.Router;
import io.vertx.ext.web.handler.*;
import io.vertx.ext.web.sstore.LocalSessionStore;

public class MainVerticle extends AbstractVerticle {

    @Override
    public void start(Promise<Void> start) {

        // Dummy database, real world workloads
        // use a persistent store or course!
        final InMemoryStore database = new InMemoryStore();

        // create the webauthn security object
        WebAuthn webAuthN = WebAuthn.create(
                        vertx,
                        new WebAuthnOptions()
                                .setRelyingParty(new RelyingParty().setName("Vert.x Demo Server")))
                // where to load/update authenticators data
                .authenticatorFetcher(database::fetcher)
                .authenticatorUpdater(database::updater);

        final Router app = Router.router(vertx);
        // parse the BODY
        app.post()
                .handler(BodyHandler.create());
        // favicon
        app.route()
                .handler(FaviconHandler.create(vertx));
        // add a session handler
        app.route()
                .handler(SessionHandler
                        .create(LocalSessionStore.create(vertx))
                        .setCookieSameSite(CookieSameSite.STRICT));

        // security handler
        WebAuthnHandler webAuthnHandler = WebAuthnHandler.create(webAuthN)
                .setOrigin(String.format("https://%s.nip.io:8443", System.getenv("IP")))
                // required callback
                .setupCallback(app.post("/webauthn/callback"))
                // optional register callback
                .setupCredentialsCreateCallback(app.post("/webauthn/register"))
                // optional login callback
                .setupCredentialsGetCallback(app.post("/webauthn/login"));

        // secure the remaining routes
        app.route("/protected/*").handler(webAuthnHandler);

        // serve the SPA
        app.route()
                .handler(StaticHandler.create());

        vertx.createHttpServer(
                        new HttpServerOptions()
                                .setSsl(true)
                                .setKeyStoreOptions(
                                        new JksOptions()
                                                .setPath("cert-store.jks")
                                                .setPassword(System.getenv("CERTSTORE_SECRET"))))

                .requestHandler(app)
                .listen(8443, "0.0.0.0")
                .onFailure(start::fail)
                .onSuccess(v -> {
                    System.out.printf("Server listening at: https://%s.nip.io:8443%n", System.getenv("IP"));
                    start.complete();
                });
    }
}
```

### Links

* https://github.com/pmlopes/authenticatecon
* https://vertx.io/
* https://github.com/vert-x3/vertx-auth/tree/master/vertx-auth-webauthn
