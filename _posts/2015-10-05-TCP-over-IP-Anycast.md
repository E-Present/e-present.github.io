---
layout: post
title:  "TCP over IP Anycast - Pipe dream or Reality?"
date:   2015-10-05 22:58:10
categories: transport
permalink: /transport/TCP-over-IP-Anycast
---

LinkedIn is committed to improving the performance and speed of our products for our members. One of the major ways we are doing that is through our Site Speed initiative. While our application engineers work to improve the speed of mobile and web apps, our performance and infrastructure teams have been working hard to deliver bytes​ faster​ to our members. To​ this end, LinkedIn heavily utilizes PoPs for dynamic content delivery. To route end-users to the closest PoP, we recently moved our major domain, www.linkedin.com, to an anycast IP address. This post talks about why we did this, what challenges we faced, and how we overcame them.
Unicast and DNS-based PoP assignment problemsIn the unicast world, each endpoint on the internet has a unique IP address assigned to it. This means that each of LinkedIn’s many PoPs (Points of Presence) has a unique IP address associated with it. DNS is then used to assign users to PoPs based on geographical location. 

In an earlier blog post, we talked about the inefficiencies of using DNS for assigning users to PoPs. We observed that about 30% of users in the United States were being routed to a suboptimal PoP when using DNS. There were two major reasons for a wrong assignment: 

	* DNS assignment is based on the IP address of the user's DNS resolver and not the user's device. So, if a user in New York is using a California DNS resolver, they are assigned to our West Coast PoP instead of the East Coast PoP.
	* The database used by DNS providers for converting an IP address to a location might not be completely accurate. Their country-level targeting is generally much better than their city-based targeting.

Anycast: What and WhyWith anycast, the same IP address can be assigned to n servers, potentially distributed across the world. The internet's core routing protocol, BGP, would then automatically route packets from the source to the closest (in hops) server. For example, in the following illustration, if a user Bob wants to connect to an anycast IP 1.1.1.1, its packets would be routed to PoP A because A is only 3 hops away from Bob, while all other PoPs are 4 hops or more. 
![](http://e-present.github.io/images/TCP-IP-Anycasr1.png)
 

Anycast’s property of automatically routing to the closest PoP gives us an easy solution to our PoP assignment problem. If LinkedIn assigned the same IP to all its PoPs, then: 

	* We would not need to rely on DNS-based geographical assignments (DNS would just hand out that one IP)
	* None of the problems associated with DNS-based PoP assignments would arise
	* Our users would get routed to the closest PoP automatically

Anycast promise: Too good to be true?Given the great properties of IP anycast, why does most of the internet still use unicast? Why have other major web companies not used anycast in an extended manner? Asking these questions gave us some disheartening news. 

Internet InstabilityAnycast has historically not proven to be useful for stateful protocols in the internet because of the inherent instability of the internet. We can explain this with an example. In the following figure, user Alice is 3 hops away from both server X and server Y. If Alice's router does per-packet load balancing, it might end up sending Alice's packets to both servers X and Y in a round-robin fashion. This means that Alice's TCP SYN packet might go to server X, but her HTTP GET request might go to server Y. Because server Y doesn't have an active TCP connection with Alice, it will send a TCP reset (RST) and Alice's computer will drop the connection and will have to restart. Most routers now do a per-flow load balancing, meaning packets on a TCP connection are always sent over the same path, but even a small percentage of routers with per-packet load balancing can cause the website to be unreachable for users behind that router. 
![](http://e-present.github.io/images/TCP-IP-Anycasr2.png)
 

Even with per-flow load balancing, another problem is that if a link on the route to server X goes down, packets on any ongoing TCP connection between Bob and X will now be sent from Bob to Y, which will cause Y to send a TCP RST again. 

This is why anycast has popularity in usage for stateless protocols like DNS (which is based on UDP). But recently, some popular CDNs have also started using anycast for HTTP traffic. This gave us hope, because their TCP connection with end users would last for a similar duration as LinkedIn's TCP connection to our end users. A Nanog presentation in 2006 claimed that anycast works. So, to validate the assumption that TCP over anycast in the modern internet is no longer a problem, we ran a few synthetic tests. We configured our U.S. PoPs to announce an anycast IP address and then configured multiple agents in Catchpoint, a synthetic monitoring service, to download an object from that IP address. Our web servers were configured to deliberately send the response back slowly, taking over a minute for the complete data transfer. If the internet was unstable for TCP over anycast, we would observe continuous or intermittent failures when downloading the object. We would also observe TCP RSTs at the PoPs. But even after running these tests for a week, we did not notice any substantial instability problems! This gave us confidence to proceed further. 

Fewer hops != Lower latencyThere were two problems with proceeding forward and actually using anycast. First, our tests were on synthetic monitoring agents and not real users. So, we couldn’t  say with high confidence that real users would not face problems over anycast. Second, and more importantly, anycast's PoP assignment might not be any better than DNS. This is because with anycast, users are routed to the closest PoP in number of hops, and not in latency. With anycast, a 1-hop cross-continental route with 100ms latency might be preferred over a 3-hop in-country route with 10ms latency. But, is this a theoretical problem or does this really happen in the internet? We ran additional catchpoint tests, but they were inconclusive, so we decided to brainstorm a real user-based solution. 
![](http://e-present.github.io/images/TCP-IP-Anycasr3.png)
 

RUM to the rescueIn a previous blog we talked about how we instrumented Real User Monitoring, or RUM, for identifying which PoP a user connects to (see the section "Which PoP served my page view?”). Taking inspiration from that solution, we devised a RUM-based technique to run experiments to find out if anycast would work for our users. 

Specifically, we did the following: 

	1. We configured all our PoPs to announce one IP address (a global anycast IP address) and configured a domain "ac.perf.linkedin.com" to point to that IP address.
	2. After a page is loaded (load event fired), an AJAX request is fired to download a tiny object on ac.perf.linkedin.com.
	3. Because the IP address is anycast, the request would be served by the PoP that is closest to the user in terms of number of hops.
	4. While responding, PoP adds a response header that uniquely identifies it.
	5. RUM reads that header and uses it to identify which PoP served the object over the anycast IP. Thus, we know which PoP would be closest to this particular end-user IP address over anycast.
	6. RUM appends PoP information to the rest of the performance data and sends it to our servers.

Through offline processing, we aggregate this data to find out, for a given geography, what percentage of users would be routed to the closest PoP (in terms of latency) over anycast. Note that we know which is the closest PoP in latency through the work explained in the earlier blog (see the section "PoP Beacons in RUM"). 

Global Anycast Results
Region/Country
DNS % Optimal 
Assignment
Anycast % Optimal 
Assignment
Illinois
70
90
Florida
73
95
Georgia
75
93
Pennsylvania
85
95
New York
77
74
Arizona
60
39
Brazil
88
33

While good for many U.S. states, the global anycast experiment also showed worse results for a few regions (for example, Brazil, Arizona, and New York). It looks as if either our peers, or some transit provider, had routing policies that made users in Brazil see Singapore as a closer PoP in terms of hops. Clearly, this would not work. 

One solution was to discover the problematic ISPs and ask them to fix their routing. This would have been a complex, arduous process without guaranteed results. So we devised a different solution. We noticed that: 

	* DNS-based geographical assignments are fairly accurate at the continent level. For example, a user in North America would usually be assigned a PoP in North America (though not always the optimal within North America).
	* Our global anycast results showed cross-continent problems. But within the continent, PoP assignments were fairly good.

Regional AnycastWe then decided to try a "Regional Anycast" solution. The regional anycast solution would work as follows: 

	* All PoPs in the same continent would get the same anycast IP address.
	* PoPs in different continents would get different anycast IP addresses.
	* We would use DNS-based geographical load balancing to hand out the continent-specific anycast IP for acpc.perf.linkedin.com.
	* We would repeat the previous experiment, but with RUM downloading the object over the acpc.perf.linkedin.com domain.

Specifically, we had 3 anycast IPs, one for each of the following regions: the Americas, Europe/Africa, and Asia. 

Upon running a similar RUM experiment, we found that the regional anycast variant didn’t have the problems seen with global anycast. Based on the results, we decided to start using the regional anycast solution. 
![](http://e-present.github.io/images/TCP-IP-Anycasr4.png)
 

RampWe first did a pilot test where all of the U.S. was slowly ramped on “Regional Anycast” over the course of few days and monitored for any anomalies. The pilot test results are shown in the following graph, where Y-axis is the percentage of optimal PoP assignment. As we slowly ramped anycast, we can clearly see that many U.S. states saw improvement in the percentage of traffic going to the optimal PoP. 
![](http://e-present.github.io/images/TCP-IP-Anycasr5.png)
 

We ramped U.S. on regional anycast earlier this year, and the overall suboptimal PoP assignment dropped from 31% to 10%. While this is a significant gain, we are still investigating why the remaining 10% are not optimally assigned. 

Final ThoughtsCurrently, we have ramped North America and Europe on regional anycast and are carefully evaluating anycast for the rest of the world. 

We do want to emphasize that anycast is not a silver bullet solution for this problem. It seems to resolve, to some extent, the inefficiencies with DNS-based PoP assignment. But it also has suboptimal assignment (in most geographies, assignment is <100% optimal), albeit for a smaller set of users. Similarly, while anycast simplifies our GLB pools, it introduces more complexity when we need to shed load from a PoP. 

Acknowledgements
This project has had major contributions from many people across many teams (Performance, Network Operations, Traffic SRE, Edge Perf SRE, GIAS and more). Thanks to everyone involved in our Anycast Working Group including Shawn Zandi for his help in the initial design; Weilu Jia and Jim Ockers for working hard to drive the project to its completion; Stephanie Schuller and Fabio Parodi for helping run the project smoothly; Sanjay Dubey for the inspiration; Naufal Jamal, Thomas Jackson, Michael Laursen, Charanraj Prakash, Paul Zugnoni, and many others. 
