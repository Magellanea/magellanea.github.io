---
layout: post
title:  "Distributed Key Value Store Hack"
date:   2015-06-15 23:56:21
comments: true
---

I decided to implement a distributed key value store as a first hack for the following reasons:

1. Data stores are something interesting!.
2. There are many papers related to that topic - like that of [Amazon Dynamo](http://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)
3. The concept itself will help me learn many things related to distributed systems.
4. There's an implementation of a similar service that can aid when stuck. For example [Voldermort](https://github.com/voldemort/voldemort).
5. A natural choice is to implement that either using [Apache Zookeeper](https://zookeeper.apache.org/) or build everything from scratch using a shiny language i.e. [Go](https://golang.org/), or [Erlang](http://www.erlang.org/)

In the next post I will explain my choice for point 5.
I hope it will be fun!.