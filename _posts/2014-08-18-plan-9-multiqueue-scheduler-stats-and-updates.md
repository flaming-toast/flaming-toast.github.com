---
layout: post
title: "Plan 9 multi–queue scheduler: stats and updates"
description: ""
category: 
tags: []
---
{% include JB/setup %}

After much bug and bottleneck hunting, I've finally finished implementing the MQS scheduler I originally described [here](http://flaming-toast.github.io/gsoc14/2014/05/17/a-multi%E2%80%93queue-scheduler-for-plan-9).

Here's a brief summary of changes made to the original 9atom scheduler, which used a single, global run queue to distribute processes to CPUs.

### Per-cpu run-queues
Each CPU runs the scheduler algorithm with its own local run queue, as described in the previous post. In order to keep these run queues balanced, fork balancing is
done with every newly created process and load balancing operations are done periodically.

### CPU 2d torus configuration
The new scheduler arranges all CPUs in a given system in a torus-esque configuration, 
where each CPU is connected to N neighbors, depending on the dimension of the torus.
What this basically means is that each CPU's load balancing operations
are limited to a set of neighboring CPUs only, thereby limiting run queue contention.
This prevents any single idle CPU from being targeted for push migrations.
The number of neighbors was set to 3 in the current implementation, 
although ideally it should be dynamically set depending on the number of CPUs available.

### Load balancing and push migration operations
If loads across all CPUs become too imbalanced ("imbalance" defined later, below), the load balancer will kick in and attempt to 
level out the run queues. A push migration scheme was chosen, which means that overloaded CPUs will attempt
to donate processes to idling CPUs to distribute load. We hypothesized that a push migration scheme would 
fare better with run queue lock contention compared to stealing (i.e. in a steal scheme, if one CPU is overloaded and numerous are idle, they will all contend for that CPU's run queue).
During stress testing, I found that the load balancer kicked in more frequently because of idling CPUs rather than load imbalances.

The load balancer checks for an "imbalance" every 500 ticks, about 0.5 ms. While stress testing the scheduler,
it was found that aggressive load balancing checks produced less fluctuations in kernel compile times (i.e., they
are more consistent from one trial to another). More on this later! 

Push migration operations primarily targeted idle CPUs. If a given CPU is 1) currently running a process and 2) has ready processes on its run queue, it will push
as soon as an idle neighbor is found. If there are no idle neighbors, the load balancer will check if a neighbor meets 
a percentage difference (currently set to 25%) in load. (That is, if a neighbor has n% less load than me, I will give it a process.)

### Stress tests and results
The schedulers were tested by running numerous, consecutive kernel compiles since such an activity spawns many processes and 
produces a good amount of I/O activity. They were run on a 4-core machine.

Average kernel compile times over 100 trials (running `time mk 'CONF=cpud'`):  
**Old 9atom scheduler**:  
_avg_: 5.466 seconds (real time)  
_min_: 4.89  
_max_: 7.4  
_std deviation_: 0.52, _variance_: 0.27

**New mqs scheduler**:  
_avg_: 5.220 seconds (real time)  
_min_: 5.11  
_max_: 5.73  
_std deviation_: 0.09, _variation_: 0.008

Below are graphs to better visualize the results of 100 consecutive kernel compilations.  
The first graph is a plot of each trial (time in seconds), the second is a bar graph that counts how
often a kernel compilation completed within a given time. 

<p align="center">
  <img src="/assets/images/2014-08-18/mqs-compile-line-graph.png"/>
</p>
<p align="center">
  <img src="/assets/images/2014-08-18/mqs-compile-bar-graph.png"/>
</p>

The new scheduler, with the push migration load balancer, is remarkably consistent, with very little variation
in compilation completion times. However, there are times where it performs a bit slower than the old scheduler.
The old scheduler has the winning minimum time of 4.89 seconds, but large fluctuations in time occurred frequently.

### Setbacks and future work
The new scheduler is functional, and project code is publicly available on Bitbucket [here](https://bitbucket.org/flaming-toast/mqs-nix), on branch **stable-final**.
Unfortunately, because of limited resources, we were only able to test the scheduler on a 4-core machine. It would
be very interesting to see how the schedulers compare on larger machines of 12-cores or more. With larger machines,
I am willing to bet that run queue contention would be much more visible.
In any case, I will be continuing to work on it over the Fall; hopefully we'll be able to procure a larger machine to continue our work. :-)

