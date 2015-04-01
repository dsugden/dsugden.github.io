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

It is common to develop distributed applications locally, and manage cloud deployment and configuration management later on, and sometimes at great cost. 


####How does conductr help Ops

ConductR enables the "elastic" part of the Reactive Manifesto. It allows Ops to scale up or down according to load without any service interruption. We will explore how this looks in an upcoming post here, stay tuned. 


####How does conductr help Devs

As a developer, I'm interested in coding for the cloud right up front. I'm also not interested in having to know ansible/chef/puppet etc. or manually deploying my app to AWS. I want to be able to stage my app and ensure correctness and performance as easily as possible. This is the conductR promise. Lets see how that looks.






