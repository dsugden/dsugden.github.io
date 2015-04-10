---
layout: post
title: Typesafe's ConductR from a developer's seat
---

This post will present a first glimpse into developing distributed applications with the Typesafe stack and, in particular, with an eye to managing the deployment, upgrade and life-cycle with [conductR](http://typesafe.com/products/conductr).

Put simply, ConductR is a tool that enables devs and ops roles to manage distributed applications.

There is more to it than that, and it's cool tech. Go read the [white paper](http://info.typesafe.com/COLL-20XX-ConductR-WP_LP.html?lst=WS&lsd=COLL-20XX-ConductR-WP&_ga=1.64711343.1443869017.1408561680), its worth your time.

###Distrubuted Applications?

Applications that need to be resilient and responsive under load need to scale out. Akka cluster, play/spray/akka-http services are commonly distributed across vms. Ops requirements are different, and obviously more complex with distributed apps.

Distributed apps need to be deployed and upgraded. They have hardware requirements. They have dependencies and perhaps need to be able to communicate within a cluster. These apps must be configured with network addresses/ports, they must be be able to specifiy hardware level requirements.

Existing tools like ansible, chef, puppet and salt are helpful, Conductr does not replace these, but enables application level management, and supports these tools with a REST api.

ConductR enables the "elastic" part of the Reactive Manifesto. It allows Ops to scale up or down according to load without any service interruption. We will explore how this looks.

###How does ConductR help Devs

As a developer, I want to be able to develop my distributed apps both **locally** and **staged in a cloud**. For example, in an akka cluster, the deploy order matters (seed nodes). Managing the configuration for clustered apps is not trivial. Nor is managing dependencies in apps distributed for scale, ie: load balancing clusters.

I also want an easy way to resolve the URLs if any dependencies I may have on other processes in the cluster.

This is the conductR promise. Lets see how that looks.

###What's the stuff in ConductR

Start with a collection of vms in the same network. ConductR DOES NOT help with this, and wasn't intended to. Use ansible/chef/puppet/salt or whatevs to create your network of nodes. 

ConductR does require on each node:
* Debian based system (recommended: Ubuntu 14.04 LTS)
* Oracle Java Runtime Environment 8 (JRE 8)
* Python 3.4 (supplied with Ubuntu 14.04)

Then you will install Conductr and a proxy (more on why later) and on each of these nodes.

Finally you will install a CLI on one or more, to admin it.

An **Application** in ConductR is a collection of one or more **Bundles**. The developer decides what bundles make up an Application, and then aggregates them with a configuration attribute ("system").

Each Bundle can contain one or more **Components**, typically just one. This represents a process in ConductR's lifecycle management terms.

When you package your Application, a ZIP file will be created for each bundle, containing a manifest.

Here is an example of bundle configuration with **one** Component:

<script src="https://gist.github.com/dsugden/af50f5bcd7eb0398d657.js"></script>

This will result in the following **bundle.conf** manifest that will be included in your .zip artifact:

<script src="https://gist.github.com/dsugden/480c6e5371737641bdc6.js"></script>

ConductR is only offered to Typesafe subscribers. So, go get yours.

So lets take a look at how a developer would get started building Conductr Bundles.

You'll need the [ConductR SBT plugin](https://github.com/sbt/sbt-typesafe-conductr) to get going.

To help you out, I've create a repo with 4 different examples of using ConductR: https://github.com/dsugden/conductrR-examples

The repo contains a Vagrantfile to bring up a 4 node ConductR cluster. Read the repo instructions.

This repo is an SBT Project with four subprojects:
 
1. Single MicroService
⋅⋅* No Dependency, Stateless microservice
2. Akka Cluster Front
⋅⋅* Belongs to an akka cluster, with an additional http component. Delegates work to Akka Cluster Back Nodes
3. Akka Cluster Back
⋅⋅* Belongs to an akka cluster, does work for Akka Cluster Front Nodes  
4. Play Project with dependency
⋅⋅* Simple Play app with a dependency on Single Microservice

I've chosen to use subprojects here, it would also make sense to structure your Bundle as a single SBT project.

###Configuration

One of the painful parts of writing clustered apps that can also be run (as they are developed) locally, is managing this dual configuration.

For example, akka endpoints:

<script src="https://gist.github.com/dsugden/c0efa5cf3f2086fd64d7.js"></script>

Now, you are deploying this akka app to the cloud, and either this file must be tokenized, or the boot class take args for some Ops script to supply the actual IPs and ports.

ConductR addresses this with it's **Endpoint** configuration declaration:

<script src="https://gist.github.com/dsugden/c74030b8ff739be10394.js"></script>

When this bundle is run by ConductR, two system env properties are created called **SINGLEMICRO\_BIND\_IP** and **SINGLEMICRO\_BIND\_PORT**.

These are available to your app, both in application.conf:

<script src="https://gist.github.com/dsugden/c5700168678f5b2719e0.js"></script>

and programatically:

<script src="https://gist.github.com/dsugden/92514ff556e9c2435de0.js"></script>

If you need these env properties passed in to your app's main as args, use the **startCommand** attribute

<script src="https://gist.github.com/dsugden/50eb241ca7ad85e7cf69.js"></script>

Voila!  Now, determining exactly which IP address is *no longer a developer's configuration problem*. That problem is now handled by OPs or ConductR itself. ConductR will know this information at runtime, and will pass it along.


###What about Akka Cluster seed nodes? 

ConductR handles akka clusters in the following way:

For any bundles that wish to join the same cluster, this aggregation happend with the "system" attribute:

<script src="https://gist.github.com/dsugden/cd3bd98f2a7cad4d0794.js"></script>

For Bundles sharing the same **system** attribute and intersecting **Endpoints**, ConductR guarantees that only one
will be starting at any given time.

This removes the need to specify seed nodes in configuration: the *first* bundle that is run designates it's containing node as the defacto seed. All other subsequent nodes that are started will be passed the IP of this first **seed** node.

In order to facilitate this two actions are required by the developer:

<script src="https://gist.github.com/dsugden/61e7213c4b6e3925b509.js"></script>

The **akka-remote** attribute must be specified as above.

If you don't care about the port:

<script src="https://gist.github.com/dsugden/be4d65813c7e0af3d1e1.js"></script>

And, the very first call in your boot code must be :

<script src="https://gist.github.com/dsugden/62ae88dce9af12c0e6b6.js"></script>

This results in: 

<script src="https://gist.github.com/dsugden/b46d2522c965ecd280ac.js"></script>

from your **application.conf** getting rewritten in sys.props to the appropriate cloud IP, eg:

<script src="https://gist.github.com/dsugden/c46ed624e72036263c16.js"></script>

##sbt-typesafe-conductr

IMO, one of the most outstanding features of the plugin is the ability to deploy and manage an app from an sbt console.

eg.

```
$sbt
$project singlemicro
```

Now we need to tell the plugin where ConductR is living:

```
conductr controlServer 192.168.59.103:9005
```

Once this is done, you can build a distribution, load it up to ConductR cluster, run , stop and unload.

```
bundle:dist
conductr load <space and tab will give you the most recent bundle>
conductr start <space and tab will give you the most recently loaded bundle>
conductr stop 
conductr unload 
...
```

This is meant for staging probably not something you'll be doing it production, and it beats scp'ing , ssh'ing etc.

Under the covers, this sbt plugin is just using the same REST API you could use from an Ops script.


###Visualizer

The folks on the CondictR team have done a great job in imagining a useful visual console for your ConductR cluster.

The console is served by a bundle called **Visualizer** that ships with ConductR. You just have to load it up.

Which brings us to **very useful** ConductR CLI.

This is something I chose to only install on one of my ConductR nodes, as you only need to interact with one node to be interacting with the whole cluster.









































































        























