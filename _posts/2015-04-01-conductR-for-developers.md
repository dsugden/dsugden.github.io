---
layout: post
title: Typesafe's conductR from a developer's seat
---

This post will present a first glimpse into developping distributed applications with the Typesafe stack and, in particular, with an eye to managing the deployment, upgrade and life-cycle with conductR.

Typesafe has provided us with early alpha access to an exciting addition to it's ecosystem: [conductR](http://typesafe.com/products/conductr)

Put simply, conductR is a tool that enables devs and ops roles to manage clustered applications.

There is a lot more to it than that, and it's cool tech. Go read the [white paper](http://info.typesafe.com/COLL-20XX-ConductR-WP_LP.html?lst=WS&lsd=COLL-20XX-ConductR-WP&_ga=1.64711343.1443869017.1408561680)

####Cluster?

Applications that need to be resilient and responsive under load need to scale out. Akka cluster, play /spray / akka-http services are commonly distributed across vms.

####What is difficult about managing apps in a cluster?

Distributed apps need to be deployed, and upgraded. They have hardware requirements. They have dependencies and need to be able to communicate within the cluster.











