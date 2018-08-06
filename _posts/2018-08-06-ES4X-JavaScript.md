---
layout: post
title: GraalVM **THE** runtime for your JavaScript server apps!
date: 2018-08-06 16:00:00 GMT
draft: false
---

During this year *Face-2-Face* meeting of the [Eclipse Vert.x](https://vertx.io) project, I've presented a [small project](https://reactiverse.io/es4x/) I was working on for a few weeks. This project, which later became known as [**ES4X**](https://reactiverse.io/es4x/), had the goal to bringing the **JavaScript** polyglot side of **Vert.x** a bit more up to date, and to make JavaScript developers at home when using Vert.x as the runtime. The goal was simple:

> The goals of this project is to offer a NPM development approach first, instead of a java project based approach. This should make the entry level for JavaScript developers quite low, if they are coming from a nodejs background.

Flashforward a few months later and [Oracle](https://oracle.com) open sourced the project [GraalVM](http://http://www.graalvm.org/). The project quickly got my attention and intrigued me with it's **Polyglot** aspect. After going through the [docs](http://www.graalvm.org/docs/reference-manual/polyglot/) it became obvious that **ES4X** should have support for this.

I was faced with the decision:

* (A) Abandon all the current `[Open|Oracle]JDK` users in favour of `GraalVM`
* (B) Try to get it working on both

As a developer I like a good challenge, so I went with option `B`. After many commits and rollbacks I believe the project got some simple abstractions that allow both: `JDK >=8` and `GraalVM` to run the same code without the need of recompilation, a true, *write once run everywhere* if I may.

Of course this didn't come out with out many support questions to the very friendly and helpful Graal team.

So we're now at `release candidate 5` and all the issues I've reported and asked have been fixed and I am happy to announce that we can run **ES4X Vert.x** application on Graal without issues. In order to verify that things are fine I've implemented the [Techempower Web Framework Benchmark](https://www.techempower.com/benchmarks/) to see if Graal and ES4X would live to the expectations. May said that Graal was [fast](https://www.forbes.com/sites/oracle/2018/04/18/graalvm-1-0-gives-developers-a-speedy-polyglot-runtime-and-helps-twitter-save-money) but we always need to double check ;-)

## The Benchmark

In order to quickly compare how things work I've took the [current](https://github.com/TechEmpower/FrameworkBenchmarks/tree/12813e17af4d841cea4c6d5f017eabc1e57b0611/frameworks/JavaScript/nodejs) `nodejs` implementation **AS IS** and run the benchmark:

|                  | json    | db     | queries | fortunes | updates | plaintext |
| ---------------- | ------- | ------ | ------- | -------- | ------- | --------- |
| requests in 1min | 4485775 | 351199 | 545688  | 410034   | 207947  | 5803708   |
| requests in 1min | 4581458 | 424491 | 127459  | 443958   | 45839   | 8862879   |
| requests in 1min | 4584105 | 466896 | 66057   | 475611   | 23404   | 8841142   |
| requests in 1min | 4636149 | 509371 | 43483   | 501849   | 15613   | 8831023   |
| requests in 1min | 4623258 | 544755 | 32366   | 501686   | 11877   |           |
| requests in 1min | 4577783 | 544863 |         |          |         |           |

* For details on the tests see the [techempower site](https://www.techempower.com/benchmarks/#section=code).

And then I've [implemented](https://github.com/reactiverse/es4x/tree/develop/examples/techempower) the ES4X version of the benchmark. You will have to note that in this implementation there are no trickery to make things go faster, for example we do parse the full request and not have a hack to extract parameters from the request path; we use the `Router` object a end user would, instead of a customized function, etc... Sadly the template engines in Vert.x have some issue (that will be fixed soon) preventing then to generate the proper JS code and we need to interact with Java from the JavaScript directly.

When the implementation above is run I've got:

|                  | json    | db      | queries | fortunes | updates | plaintext |
| ---------------- | ------- | ------- | ------- | -------- | ------- | --------- |
| requests in 1min | 7200621 | 745994  | 1336536 | 396406   | 671466  | 24794384  |
| requests in 1min | 7193445 | 862764  | 363914  | 398139   | 204794  | 23283488  |
| requests in 1min | 7959525 | 953472  | 189393  | 434055   | 134405  | 23173456  |
| requests in 1min | 8424036 | 1067111 | 133491  | 481231   | 110602  | 23203120  |
| requests in 1min | 8666016 | 1215485 | 101816  | 496435   | 40168   |           |
| requests in 1min | 8717181 | 1347345 |         |          |         |           |

**Notes:** The start command for this run was:

```sh
~/graalvm-ce-1.0.0-rc5/bin/java \
    -server                                           \
    -XX:+UseNUMA                                      \
    -XX:+UseParallelGC                                \
    -XX:+AggressiveOpts                               \
    -Dvertx.disableMetrics=true                       \
    -Dvertx.disableH2c=true                           \
    -Dvertx.disableWebsockets=true                    \
    -Dvertx.flashPolicyHandler=false                  \
    -Dvertx.threadChecks=false                        \
    -Dvertx.disableContextTimings=true                \
    -Dvertx.disableTCCL=true                          \
    -jar                                              \
    target/techempower-benchmark-1.0.0-fat.jar        \
    --instances                                       \
    `grep --count ^processor /proc/cpuinfo`
```

## The Results

These results are **amazing**, it shows that running your server side code with [vert.x](https://vertx.io) and [ES4X](https://reactiverse.io/es4x/) will give you **at least** **157%** performance boost, up to **708%**. The only test that was worse was the `fortune` test, and, as explained this is due to the fact that currently the generated API is not the best.

Here are the final results as `ES4X / NodeJS`:

| ES4X / NodeJS | json    | db      | queries | fortunes | updates | plaintext |
| ------------- | ------- | ------- | ------- | -------- | ------- | --------- |
|               | 1.605   | 2.124   | 2.449   | 0.966    | 3.229   | 4.272     |
|               | 1.570   | 2.032   | 2.855   | 0.897    | 4.468   | 2.627     |
|               | 1.736   | 2.042   | 2.867   | 0.913    | 5.743   | 2.621     |
|               | 1.817   | 2.095   | 3.067   | 0.959    | 7.084   | 2.627     |
|               | 1.874   | 2.231   | 3.146   | 0.990    | 3.382   |           |
|               | 1.904   | 2.472   |         |          |         |           |

So if you're doing server side **JavaScript**, think again when bluntly you nod along with *node is fast*!
