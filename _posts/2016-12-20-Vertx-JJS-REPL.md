---
layout: post
title: Vert.x and JJS REPL
date: 2016-12-20 13:00:00 GMT
---

One of the productivity features of platforms such as Node.JS is their REPL. Vert.x runs on the JVM however it can still use `JJS` the `JavaScript` REPL.

## How it works

Given that the bootstraping of the project was done with the [nashorn experiment](https://github.com/pmlopes/vertx3-nashorn.next) any project can be run on the `REPL`.

Assuming that the project has already been built as a fat-jar with a [pom.xml](https://github.com/pmlopes/vertx3-nashorn.next/blob/master/examples/jjs/pom.xml) all it is needed to do is:

1. Start the REPL: `jjs -cp jarfile.jar`
2. Load vert.x into the engine `load('classpath:vertx.js');`
3. Load the application: `load('main.js');`
4. Profit!

<amp-youtube data-videoid="3tbqwWvxN98" layout="responsive" width="480" height="270"></amp-youtube>
