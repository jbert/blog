---
title: "Unbalanced Load"
date: 2017-11-26T14:26:44Z
draft: true
tags: ["cocoon", "cloud", "investigation"]
description: "Random load balancing seems simple, but we found an interesting failure mode."
---

Random load balancing seems simple, but we found an interesting failure mode.

## Motivation

Load balancing is a cloud computing technique needed when you grow a service
out beyond a single server. When a client request is received, something has
to decide to which server should handle it. The goal is to evenly utilise your
servers in order to minimise the number of servers you need.

There are various different approaches to doing this. I'd like to explore some
of the choices we made, the reasoning behind them and the consequences.

## Background

There are two characteristics about the requests this service receive which
are important:

1) The requests were not made by a human. A failure-and-retry was acceptable.

2) The requests were very long lived - perhaps 2-24 hours long.

The first meant that our load balancer could be truly ignorant, it didn't even
need knowledge of which servers were available. And using the maxim "Do the
simplest thing which could possibly work", we decided to simply randomly
allocate a server for each incoming request.

## The symptom

After running the service in test and staging for a while, we took a closer
look at the server utilisation for capacity planning purposes. And found this:

![The Problem](/img/load-balancing-problem.png)

The interesting parts of this graph are the two "dips" which correspond to
times when we deployed new code - restarting each server in turn to pick up
the new codebase.

Looking at this, my first thought was that our simple-as-possible discover
service was broken. The sessions must be somehow 'sticky' to an server, as
we can clearly see that after the second dip, the red and yellow servers still
have a disproportionate share of the client load.

The developer most familiar with the service soon enlightened me. The
imbalance was usual on a code deploy and was related to the order in which the
servers were restarted. Since the same order was used on each deploy, the
same imbalance was present. If we went long enough without deployment of an
updated codebase, it would slowly balance out over time.

## The problem

It is perhaps obvious in hindsight, but it was fun thinking through what was going on.

When an server is restarted, all its clients are bounced off and forced to
reconnect. In doing so, they are evenly shared out among the other servers
by the random allocator.

Because the requests are so long lived, the client connections then *stay
there* for the 2+ hour duration.

This means that the last server to be restarted has effectively zero load,
since all client connections are being handled.

OK, this is a problem. But how much of a problem? How much extra load will the
most-loaded instance have to take? How will the imbalance change as we
increase the number of servers? i.e. Do we need to fix this now....?

## Analysis

We could with the simple case where we have N servers, all evenly loaded, then
restarted in turn. When a server is restarted, it's load goes to zero and it's
current share is distributed to the others. 

Doing this analytically is a little annoying. But we can usefully observe that
the spread of load will be linear depending on the restart order. The most
recently started will have a load of zero. We can picture this as a 'triangle'
of load with the area underneath it being the same as an evenly-distributed
rectangle of load. Since a triangle of the same area must be twice as high
as a rectangle, we can see the first-restarted server gets twice the load it
would in a well-balanced setup - and that this doesn't change as we add more
servers.

![Linear load](/img/unbalanced-linear.png)

This means we'd have to over-provision by 2x to get the same utilisation
figures - not a great outcome.

## Designing a solution

So the "simplest thing which cloud possibly work" was a bit too simple. We
need a better approach.

It seems clear that the load balancer needs to direct incoming requests to a
server which has a comparatively low number of connections.

But a naive "send to the least loaded" approach has a potentially fatal flaw
in practice. A server which - for whatever reason - goes "rogue" and is
erroring on every request will always have a low load and so will attract
every incoming connection, leading to a complete outage.

To hedge against this, we want to have a pool of servers to choose from which
are likely to be lower-than-average loaded.

I at first thought we could make a choice among the pool of servers with a
load below the mean. That would guarantee that we always made some progress
towards being better balanced on every request.

Fortunately, the other two developers interested in this problem pointed out
to me that there is a common case where this re-introduces the "least-loaded
problem". If you have even load and restart one server, you have exactly one
server which is below the mean, so your pool of choices is 1.

So, we have to accept that we need a pool to choose from and - with some
distributions of load we may make a sub-optimal choice in terms of balancing.

Which leads us to sorting the servers by load and make a choice from the bottom 50%.

The 50% here is an choice of a tunable parameter. The smaller value we choose,
the 'better' our load balancer will be at choosing the least loaded, at the
cost of a greater impact on the service in the case we have a rogue server.

## Results

With some real load on the live service, we know have an improved initial load
profile following a rolling restart:

![Better](/img/better.png)

There **are** still lightly loaded servers (the last to be restarted), but
more importantly the most-loaded servers after a restart are carrying much
less than a double load, with the corresponding benefit to capacity planning.

We could tune this further (primarily by dropping the 50% parameter) but this
is a big improvement for now.
