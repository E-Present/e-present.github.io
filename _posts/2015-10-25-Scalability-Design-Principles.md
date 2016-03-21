---
layout: post
title:  "Scalability Design Principles"
date:   2015-10-25 21:00:13
categories: transport
permalink: /transport/Scalability-Design-Principles
---

## Scalability Design Principles
What different types of scalability are there, and how do you design for them? Elastisys is a spin-off from the distributed systems research group at Umeå University. As such, we have extensive knowledge and experience in designing and developing scalable distributed applications. In this post, we list some of our best tips to help software architects and CTOs design with scalability in mind.

Different Types of ScalabilityScalability is an overloaded term. What comes to your mind when you read it? Some will think of claims of linear scalability for NoSQL databases. Others will think of the elastic and scalable nature of cloud infrastructure itself. And so forth. We think of scalability exists in several dimensions:

	* Scalability of Performance. High performance is paramount to supporting many users. Intuitively, adding more capacity to an application component should increase the component’s performance. However, Amdahl’s Law states that there is a limit to how much benefit we can get. First note that it was intended for parallel computing, not distributed systems. It states that speedup of a program using multiple processors in parallel computing is limited by the sequential fraction of the program. Some problems are simply better suited for tackling with a parallel approach (consider the problem of digging a ditch versus the problem of digging a hole). Research has tried to update Amdahl’s law, other laws have been suggested, but the core lesson remains: performance does not scale perfectly linearly. We can, however, try use components designed with parallelism and asynchronism in mind. In short: work queue with notification upon completion good, synchronous connection to database bad.
	* Scalability of Availability.The CAP theorem states that a distributed system cannot simultaneously provide all the guarantees of consistency, availability, and partition tolerance. In many web-scale applications, strong consistency guarantees are often being dropped in favor of the other two. Instead, they rely on eventual consistency: at some point the system will let all users read the most recently made updates, but all will not do so immediately. This greatly reduces synchronism in our application and allows the system to remain available to all users, even in the face of network partitions — during the partition, users may see inconsistent data, but as the partition heals the consistency of the system will eventually be restored.
	* Scalability of Maintenance. Software is not just written and forgotten, it must be maintained. Likewise, servers are not just started, they must be maintained. We do that by using platforms and tools that help us monitor our application, update parts when needed, and ensure it operates properly. Of course, we want to do as much as possible of this automatically.
	* Scalability of Expenditures. Total cost of ownership (TCO) includes development, maintenance, and operational expenditures. When we design a system, we need to balance making use of existing components and developing everything ourselves. Existing components rarely fully match our needs, but whatever training and additional development required to make them fit might cost less than developing an entirely unique solution. Using industry standard technology might also make it easier to hire an expert in the future. Conversely, publishing a unique solution as open source might help us identify community members that would make great additions to our team.

Each of these are important to consider when we design our applications. Some of these aspects overlap, some even contradict each other at times. Software architects and CTOs always have to strike a balance! Let’s see these aspects distilled into design principles.

## Scalability Design Principles

	1. Avoid the single point of failure. We can never just have one of anything, we should always assume and design for having at least two of everything. This adds costs in terms of additional operational effort and complexity, but we gain tremendously in terms of availability and performance under load. Also, it forces us into a distributed-first mindset. If you can’t split it, you can’t scale it has been said by various people, and it’s very true.
	2. Scale horizontally, not vertically. There is a limit to how large a single server can be, both for physical and virtual machines. There are limits to how well a system can scale horizontally, too. That limit, though, is increasingly being pushed further ahead. Even databases are moving in that direction. Furthermore, the cost of (vertically) upgrading a server increases exponentially whereas the cost of (horizontally) adding yet another (commodity) server increases linearly.
	3. Push work as far away from the core as possible. There are several orders of magnitude more clients than servers as we move inward into the core of our application. The less work the few have to do on behalf of the many, the better. The smaller the updates we can pass to our clients, the better.
	4. API first. In addition to pushing work to the clients, view your application as a service with an API first. Clients these days are smartphone apps, web sites with JavaScript, and desktop applications. If the API does not make assumptions about which clients will connect to it, it will be able to serve all of them. And you open your service up for automation, as well.
	5. Cache everything, always. Caches are essentially storages of precomputed results that we use to avoid computing the results over and over again. This is a Godsend for scalability and performance, so we must use it.
	6. Provide as fresh as needed data. Depending on your application, users might not need the very freshest data right away. Eventual consistency leads to much better availability under the CAP theorem. If we actually need strict consistency, so be it.
	7. Design for maintenance and automation. Software needs monitoring and updates to ensure proper operation over time. As we move out of the servers are pets era and into the servers are cattle era, our mindset has to change. Do we even need to reconfigure servers anymore? Can’t we just take old ones down and replace with new ones, that have been configured offline as part of their image creation? What monitoring data is important to us? What information does it provide us with? Do not under-estimate the time and effort spent in maintaining your application. Your initial public software release is a laudable milestone, it also marks when the realwork begins.
	8. Asynchronous rather than synchronous. We already understand asynchronous communication perfectly in the physical world. We drop off a letter in the mail, and some time later, it arrives. Until it does, we convince ourselves that it is underway, oblivious to the complexity of the postal system. We only use personal couriers for very important messages. A similar approach should be taken for our applications. Did a user just hit submit? Tell the user that the submission went well, and then process it in the background. Perhaps show the update as if it is already completely done in the mean time.
	9. Strive for statelessness. While it may seem tempting to avoid inter-component communication by keeping track of certain state information in e.g. your application servers, don’t. Unless we host purely static pages, we can never get away from state information. We must make sure that state information is kept in as few places as possible, and within components made for it. Web and application servers are not, but distributed key-value stores are. Keeping it there lets you treat your web and application servers as completely replaceable instances, which is ideal from a scalability point of view since your server fleet can much more easily be modified when any server is able to handle any client request (despite the client being in the midst of a “session”).
	10. This too shall fail. Computer systems fail. Software fails. Hardware fails. Designs fail. Failure handling fails! Be prepared for failure, but spare end users from witnessing it too obviously. It reflects poorly on you, even if failure is inevitable.

Do you have more scalability design tips and insights to share? Let us know in the comments below!
