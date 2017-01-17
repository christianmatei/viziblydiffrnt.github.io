---
layout: post
title: "Choosing a Tableau Server Architecture"
date: 2016-07-21
tags: [Tableau Server]
header:
  teaser: /assets/images/architecture_key.png
---

<p align="center">
<img src="https://viziblydiffrnt.github.io/assets/images/architecture_key.png">
</p>

Depending on your organization's needs, there are a variety of ways that you can configure Tableau Server. Choosing the right architecture for your server deployment is a crucial first step and can make all the difference in your users' experience, and that of your administrator as well. So how do you know what configuration is right for you?

## Single Node or Distributed Configuration

The decision to go with a Single Node installation or a Distributed (clustered) configuration should be based around a combination of the following details:

* Total User Count (the maximum number of licensed users)
* Concurrent Users (between 5 and 10% of your total user count should be a good enough estimate)
* Your License Strategy
* The Number of Extracts vs. Live Data Connections
* The amount of load you plan to drive ([What Drives Tableau Server Load?](https://viziblydiffrnt.wordpress.com/2016/01/12/what-drives-tableau-server-load/))
* Availability expectations for your Tableau environment

So how many users do you have? Are you testing Tableau out with a select number of people in your organization or is your charter to grant access to everyone for ultimate transparency? In either case, there is a licensing strategy for all deployments and choosing the best one will also influence how you set up your configuration.

### User-Based or Core Based License

User-based licensing is exactly what it sounds like. You purchase a license for every named user on your server. This is a common license strategy because most installations start out small and then grow organically. You may only have a small handful of users to begin with but this can quickly grow to 25, 50, 100+. For these environments, a single node running 8 cores with 32GB (the bare minimum) is generally sufficient and user-based licensing is perfectly suited.

But what happens when the entire company wants to get on the server? How can you add additional users when you have a limited amount of licenses? Wouldn't it be great if you could automatically provision new users to a team that's growing rapidly? That's where core-based licensing comes in.

In contrast to user-based licensing, core-based licensing is based on the physical cores of your server's hardware. You pay based on the processor cores of your hardware and the only limit to the number of users you can add is the level of performance you can consistenly deliver. With core-based licensing, you can onboard entire teams and quickly scale to meet your company's demand. This license strategy assume a large volume of users and is most often associated with enterprise deployments.

Now that you've determined what licensing strategy is best for your deployment, it's time to think about performance. As adoption increases, the volume of content on your server is going to grow and it's quite common for there to be a perceived slow down in performance. Put simply, the demand for content (# of visualizations, extract refreshes, subscriptions being sent) has outgrown the original hardware specification and it's time to upgrade.

So should you scale up or scale out?

### Scaling Up

Scaling up means adding additional capacity within your current hardware. This could mean increasing the number of cores available to Tableau Server processes, adding more RAM to the system, swapping out hard disks for solid state drives or other improvements. Each of the above methods of scaling up comes with its own costs and some are greater than just the cost of the hardware. If you want to add additional cores to your server, you'll need to upgrade your server license to leverage them and there's a cost associated with that too. Scaling up can provide significant runway for your server and modern enterprise hardware can be upgraded several times. Eventually you'll reach the limit of what a single machine can do. So what then? That's where scaling out comes into play.

### Scaling Out

Scaling out refers to adding additional capacity by increasing the total number of machines available to your Tableau Server configuration. Tableau Server supports a distributed or clustered configuration by letting you add additional machines to spread the load across them and thus increasing the overall capacity of your environment. As you can imagine, this can lead to some pretty complex configurations but for simplicity sake I'll touch on just a couple of reasons why you might want to consider scaling out as an option:

### Additional Capacity

The most common reason I see for scaling out is to add additional capacity to Tableau Server. If you've maxed out your single node and performance is still not where you want it to be, adding a 2nd or 3rd node can provide the additional horsepower you need. Tableau Server does this through the power of Worker nodes. Worker nodes act in concert with the Primary node and can share the load of your Tableau environment.

There are several clustered configurations that you can adopt and the one you choose mostly depends on how your users interact with content on the server. If your server has a lot of extract refreshes being performed daily, consider using a dedicated worker node for background processing.

Tableau's Backgrounder process is a primary driver of load on Tableau Server (see Paul Banoub's excellent post on [All About the Backgrounder](https://vizninja.com/2016/05/05/tableau-server-all-about-the-backgrounder/) for more detail) and offloading it to a separate machine can yield benefits including:

Reduced strain on the VizQL process (which is responsible for rendering all those beautiful visualizations your team has built).
Allocating more process threads to refreshing extracts, sending subscriptions, or executing commands via tabcmd. More process threads = more simultaneous jobs = less delay for content updates.

<p align="center">
<img src="https://viziblydiffrnt.github.io/assets/images/dedicated_backgrounder.jpg">
</p>

<p align="center"><i>Example of a distributed system with a dedicated "Backgrounder" machine running 10 Background Processes</i>
</p>

### High Availability (HA)

High Availability is another major reason why I see organizations decide to scale out their servers. Tableau's [documentation](https://onlinehelp.tableau.com/current/server/en-us/distrib_ha_intro.htm) does a great job of explaining what High Availability is so I won't go into depth here but if you want to ensure that your Tableau Server is up even if one of your nodes goes down, you're going to need a distributed architecture.

The best reason to scale out is if you want to build a high-performance, highly-available Tableau Server. These types of environments aren't cheap but if you're supporting thousands of users running thousands of extract refreshes around the clock it'll give you the confidence to know that you can serve their needs.

<p align="center">
<img src="https://viziblydiffrnt.github.io/assets/images/Tableau_HA.jpg">
</p>

<p align="center"><i>Highly available configuration with external load balancer, 1 Primary, 2 Worker Nodes, 2 Background Nodes, 1 Backup Primary: 64 total cores
</i>
</p>

## Bare Metal or VMs

Now that you've chosen your configuration, it's time to decide what kind of hardware you're going to deploy it on. Will you deploy on Bare Metal or use Virtual Machines (VMs)?

The decision on whether to use physical hardware or go with virtual machines is a major one that should be thought through carefully. Virtual machines can be a great way to go as they take less time to procure and set up and are often cheaper than ordering custom hardware. A word to the wise though...if you go with VMs for Tableau Server there are some strict criteria that you'll need your system adminstrators to guarantee or managing your server's reliability will be a nightmare:

* Tableau's minimum hardware requirements assume you are using bare metal physical servers, not virtual machines. The same is true for their scalability estimates.
* Because Tableau Server is a resource-intensive and latency-sensitive application, it requires dedicated resources. Pooling or sharing of CPU and RAM across other VMs on a host machine will lead to lots of issues with your configuration. Avoid pooling and sharing at all costs.
* Make sure you get 100% dedicated CPU allocation. Tableau's minimum requirement for a Production environment is 8 **physical** cores. Tableau ignores hyperthreading so make sure when you ask for 8 cores you're really getting the full amount.
* 100% dedication of RAM is important too. It cannot be dynamic RAM either. It should be contiguous on the host server and the minimum for a Production environment is 32GB. So if you're running an HA setup with 3 machines you'll need at least 96GB of dedicated RAM from a single VM host.
* The high-latency of Tableau Server also requires fast disk access. Don't settle for network attached storage (NAS). Insist that you're given a tiered SAN to get the highest level of write speed.
* If you're running a distributed environment, you'll want the latency between your workers to be less than 10ms. A poorly tuned VM setup will lead to poor server performance and you'll feel like you wasted a lot of money on those extra cores you purchased.
* Do not use VM snapshots for backing up Tableau Server. The only supported method for backing up Tableau is to use the 'tabadmin backup' command.
* If your system admin is insisting on giving you a setup that has VMotion enabled, don't use VMs. VMotion is a technology that allows VMs to move from one physical host to another for managing performance and it is not compatible with Tableau's licensing technology. More about why this happens can be found here: [VM server vs. Physical server](https://community.tableau.com/thread/163419?start=15&tstart=0).
* Keep in mind that the above applies to all machines running Tableau Server. If you're using a clustered setup, make sure you get the same guarantees for the Worker nodes and the Primary.

## Putting it All Together

By now you should have a good idea of what type of license you're going to use, what your configuration will look like and whether or not you're going to use physical hardware or go with a slate of dedicated VMs. Every installation of Tableau Server is unique to the business cases and environment it serves. I'd love to hear about your experiences with single vs. clustered setups and if you've had challenges with VMs over bare metal (or vice versa) so please leave your stories in the comments.