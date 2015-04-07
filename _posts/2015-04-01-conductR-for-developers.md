---
layout: post
title: Typesafe's ConductR from a developer's seat
---

This post will present a first glimpse into developping distributed applications with the Typesafe stack and, in particular, with an eye to managing the deployment, upgrade and life-cycle with [conductR](http://typesafe.com/products/conductr).

Put simply, ConductR is a tool that enables devs and ops roles to manage clustered applications.

There is more to it than that, and it's cool tech. Go read the [white paper](http://info.typesafe.com/COLL-20XX-ConductR-WP_LP.html?lst=WS&lsd=COLL-20XX-ConductR-WP&_ga=1.64711343.1443869017.1408561680), its worth your time.

####Cluster?

Applications that need to be resilient and responsive under load need to scale out. Akka cluster, play/spray/akka-http services are commonly distributed across vms. 

####Whats the big deal with managing apps in a cluster?

Distributed apps need to be deployed and upgraded. They have hardware requirements. They have dependencies and need to be able to communicate within the cluster. These apps must be configured with network addresses/ports.

It is common to develop distributed applications locally, and manage cloud deployment and configuration management later on at great cost. 

When these things go wrong or take too long, it costs. $$$.

####How does conductr help Ops

ConductR enables the "elastic" part of the Reactive Manifesto. It allows Ops to scale up or down according to load without any service interruption. We will explore how this looks.

####How does conductr help Devs

As a developer, I want to be able to develop my distributed apps both **locally** and **staged in a cloud**. In an akka cluster, the deploy order matters (seed nodes). Managing the configuration for clustered apps is not trivial. Nor is managing dependencies in apps distributed for scale, ie: load balancing clusters.

I also want an easy way to resolve the URLs if any dependencies I may have on other processes in the cluster.

This is the conductR promise. Lets see how that looks.

####What's the stuff in ConductR

Start with a collection of vms that can speak to each other. ConductR DOES NOT help with this, and wasn't intended to. Use ansible/chef/puppet/salt or whatevs to create your network of nodes. 

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

```scala
lazy val singlemicro = (project in file("singlemicro"))
      .enablePlugins(JavaAppPackaging,SbtTypesafeConductR)
      .settings(
        name := "singlemicro",
        version  := "1.0.0",
        BundleKeys.nrOfCpus := 1.0,
        BundleKeys.memory := 64.MiB,
        BundleKeys.diskSpace := 5.MB,
        BundleKeys.endpoints := Map(
        "singlemicro" -> Endpoint("http", 8096, Set(URI("http:/singlemicro")))))
```
This will result in the following **bundle.conf** manifest that will be included in your .zip artifact:


```
version    = "1.0.0"
name       = "singlemicro"
system     = "singlemicro-1.0.0"
nrOfCpus   = 1.0
memory     = 67108864
diskSpace  = 5000000
roles      = []
components = {
  "singlemicro-1.0.0" = {
    description      = "singlemicro"
    file-system-type = "universal"
    start-command    = ["singlemicro-1.0.0/bin/singlemicro", "-J-Xms67108864", "-J-Xmx67108864"]
    endpoints        = {
      "singlemicro" = {
        protocol  = "http"
        bind-port = 8096
        services  = ["http:/singlemicro"]
      }
    }
  }
}
```

So lets take a look at how a developer would get started building Conductr Bundles.

You'll need the [ConductR SBT plugin](https://github.com/sbt/sbt-typesafe-conductr) to get going.

To help you out, I've create a repo with 4 different examples of using ConductR: https://github.com/dsugden/conductrR-examples

This repo is an SBT Project with four subprojects:
 
1. Single MicroService
⋅⋅* No Dependency, Stateless microservice
2. Akka Cluster Front
⋅⋅* Belongs to an akka cluster, with an additional http component. Delegates work to Akka Cluster Back Nodes
3. Akka Cluster Back
⋅⋅* Belongs to an akka cluster, does work for Akka Cluster Front Nodes  
4. Play Project with dependency
⋅⋅* Simple Play app with a dependency on Single Microservice

I've chosen subprojects to make this blog post easier, but, in the wild, it would also make sense to structure your Bundle as a single SBT project.


###Configuration

One of the painful parts of writing clustered apps that can also be run (as they are developed) locally, is managing this dual configuration.

For example, akka endpoints:

```
akka.remote {
    netty.tcp {
      hostname = "127.0.0.1"
      port = 8089
    }
  }
```

Now, you are deploying this akka app to the cloud, and this file must be tokenized for some Ops script to supply the actual IPs and ports.

ConductR address this with it's **Endpoint** configuration declaration:

```scala
BundleKeys.endpoints := Map("singlemicro" -> Endpoint("http", 8096, Set(URI("http:/singlemicroservice"))))
```

When this bundle is run by ConductR, two system env properties are created called **SINGLEMICRO_BIND_IP** and **SINGLEMICRO_BIND_PORT**.

These are available to your app, both in application.conf:

```
singlemicro {
  ip = "127.0.0.1"
  ip = ${?SINGLEMICRO_BIND_IP}
  port = 8096
  port = ${?SINGLEMICRO_BIND_PORT}
}
```

and programatically:

```scala
sys.env.get("SINGLEMICRO_BIND_IP")
```

If you need these env properties passed in to your app's main as args, use the **startCommand** attribute

```scala
BundleKeys.startCommand += "-Dhttp.address=$SINGLEMICRO_BIND_IP -Dhttp.port=$SINGLEMICRO_BIND_PORT"
```

Voila!  Now, determining exactly which IP address is *no longer a developer's configuration problem*. That problem is now handled by OPs or ConductR itself. ConductR will know this information at runtime, and will pass it along.


###What about Akka Cluster seed nodes? 

ConductR handles akka clusters in the following way:

For any bundles that wish to join the same cluster, this aggregation happend with the "system" attribute:

```scala
BundleKeys.system := "SomeAkkaClusterSystem"
```

For Bundles sharing the same **system** attribute and intersecting **Endpoints**, ConductR guarantees that only one
will be starting at any given time.

This removes the need to specify seed nodes in configuration: the *first* bundle that is run designates it's containing node as the defacto seed. All other subsequent nodes that are started will be passed the IP of this first **seed** node.

In order to facilitate this two actions are required by the developer:
```scala
 BundleKeys.endpoints := Map("akka-remote" -> Endpoint("tcp", 8084, Set.empty))
```
The **akka-remote** attribute must be specified as above.

If you don't care about the port:

```scala
 BundleKeys.endpoints := Map("akka-remote" -> Endpoint("tcp", 0, Set.empty))
```

And, the very first call in your boot code must be :

```scala
ClusterProperties.initialize
```

This results in: 

```
 akka.cluster.seed-nodes = ["akka.tcp://SomeAkkaClusterSystem@127.0.0.1:8089"]
```

from your **application.conf** getting rewritten in sys.props to the appropriate cloud IP, eg:

```
 akka.cluster.seed-nodes = ["akka.tcp://SomeAkkaClusterSystem@10.0.1.60:8089"]
```


##sbt-typesafe-conductr

IMO, one of the most outstanding features of the plugin is the ability to deploy and manage an app from an sbt console.

eg.

```
$sbt
$project singlemicro
```

Now we need to tell the plugin where ConductR is living:

```
conductr:controlServer 192.168.59.103:9005
```

Once this is done, you can build a distribution, load it up to ConductR cluster, run , stop and unload.

```
bundle:dist
loadBundle < space and tab will give you the most recent bundle>
startBundle <bundleId>
...
```

This is meant for staging probably not something you'll be doing it production, and it beats scp'ing , ssh'ing etc.




























































        
























