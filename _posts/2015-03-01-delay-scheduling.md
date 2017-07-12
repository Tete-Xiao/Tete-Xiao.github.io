---
layout: post
title: "Delay Scheduling: A Simple Technique for Achieving
Locality and Fairness in Cluster Scheduling"
date: 2015-03-01
comments: True
categories: "big-data"
description: "This is a paper motivated by the scheduling problem raised in traditional FIFO strategy in data-intensive cluster computing system. The proposed methodology is designed to get a good tradeoff point between the conflicts of fairness and data locality, which practically improve response time for small jobs by 5x in a multi-user workload and double throughput in an IO-heavy workload."
---

This is a paper motivated by the scheduling problem raised in traditional FIFO strategy in data-intensive cluster computing system.

The proposed methodology is designed to get a good tradeoff point between the conflicts of fairness and data locality, which practically improve response time for small jobs by 5x in a multi-user workload and double throughput in an IO-heavy workload.

<!--more-->

<hr class="soft"/>

### Introduction

#### Problem with Hadoop

* Data consolidation provided by a shared cluster is highly beneficial

* When enough groups began using Hadoop, job response times started to suffer due to Hadoop's FIFO scheduler

#### HFS (Hadoop Fair Scheduler)

* To solve the above raised problem, HFS is designed for two main goals:
  * __Fair Sharing__
    * Divide resources using max-min fiar shaing to achieve statistical multiplexing
  * __Data Locality__
    * Place computations near their input data, to maximize system throughput


  * To achieve the first goal, a scheduler must reallocate resources between jobs when the number of jobs changes
  * A key design question is what to do with tasks from running jobs when a new job is submitted, in order to gie resources to the new job


  * At high level, two approaches can be taken
    1. __Kill__ running tasks to make room for the new job
      * Killing reallocates resources instantly, gives control over locality for new job
      * But have drawback for wasting the work of killed tasks
    2. __Wait__ for running tasks to finish
      * Waiting doesn't waste work from killed tasks
      * But can negatively impact fairness


* __The Principal result__ in this paper
  * An algorithm based on waiting can achieve both high fairness and high data locality
  * First, in large clusters, tasks finish at __such a high rate__ that resources can be reassigned to new jobs on a timescale much   smaller than job durations
  * However, a strict implementation of fair sharing compromises locality
  * Because the job to be scheduled next according to fairness might not have data on the nodes that are currently free

* To resolve it, the fairness is relaxed slightly through __delay scheduling__
  * A job waits for a ilmited amount of time for a scheduling opportunity on a node that has data for it
  * A very small amount of waiting is enough to bring locality close to 100%
  * Delay scheduling only asks that we sometimes give resources to jobs out of order to improve data locality

### Background

#### Hadoop Distributed File System

* __Job scheduling__ at Master
  * Default Scheduler runs jobs in FIFO order, with five priority levels
  * When the scheduler receives a heartbeat indicating that a map or reduce slot is free, it scans through jobs in order of priority and submit time to find one with a task of the required type
* __Locality Optimization for Map operation__
  * After selecting a job, the scheduler greedily picks the map task in the job with data closest to the slave


### Delay Scheduling  

#### Naive Fair Sharing Algorithm

* A straight forward strategy is to assign free slots to the job with fewest running tasks
  * As long as slots become free quickly enough, the resulting allocation will satisfy max-min fairness
* __Algorithm 1: Naive Fair Sharing__

{% highlight python %}
def scheduler(...):
    while a heartbeat is received from node n:
        if n has a free slot:
            sort jobs in increasing order of number of running tasks
            for j in jobs:
                if j has unlaunched task t with data on n:
                    launch t on n
                elif j has unlaunched task t:
                    launch t on n
{% endhighlight %}


* Waiting will not have significant impact on job response time if at least one of the following conditions holds:
  1. Many jobs
  2. Small jobs
  3. Long jobs

* Data locality problems with Naive Method
  * Head-of-line Scheduling
  * Sticky Slot


#### Delay scheduling
* __Algorithm 2: Fair sharing with Delay Scheduling__

{% highlight python %}
def delay_scheduler(...):
    initialize j.skipcnt = 0 for all jobs j
    while a heartbeat is received from node n:
        if n has a free slot:
            sort jobs in increasing order of number of running tasks
            for j in jobs:
                if j has unlaunched task t with data on n:
                    launch t on n
                    j.skipcnt = 0
                elif j has unlaunched task t:
                    if j.skipcnt >= D:
                        launch t on n
                    else:
                        j,skipcnt += 1
{% endhighlight %}

* __Note__: Once a job has been skipped D tiems, we let it launch arbitrarily many __non-local tasks without resetting the skipcount__. When if manages to launch a local task again, we set its skipcount back to 0.

#### Analysis of Delay Scheduling

* __Two interesting observations to the key questions__:
  1. How much locality improves depending on D?
    * Non-locality __decreases exponentially__ with D

  2. How long a job waits below its fair share to launch a local task?
    * The amount of waiting required to achieve a given level of locality is __a fraction of the average task length and decreases linearly with the number of slots per node L__

* Observation 1:
  * The probability that a job finds at least one local task with __skipCount threshold D__ is:
$$
\begin{equation}
    Prob = 1 - (1 - p_j)^D
\end{equation}
$$
    * Where $p_j = \frac{\mid P_j \mid}{M}$, $P_j$ is the set of nodes that job j has local data left on it, and $M$ is the total number of nodes in the cluster

* As we can find that the probability decrease exponentially.
   * For example, when $p_j = 0.1$, and $D=10$, we have $prob = 0.65$ => when $D=40$, we have $prob = 0.99$

* Observation 2:
  * Once a job j reaches the head of the queue, it will wait at most $\frac{D}{S} \cdot T$ seconds before being allowed to launch non-local tasks.
    * Local tasks run faster than non-local tasks up to 2x


#### How to set D

* Suppose we wish to achieve locality greater than $\lambda$ for jobs with N tasks on a cluster with M nodes, L slots per node and replication factor R.

$$
\begin{equation}
    D \geq - \frac{M}{R} \cdot \ln{( \frac{(1 - \lambda) \cdot N}{1 + (1 - \lambda) \cdot N }  )}
\end{equation}
$$

#### Rack Locality

* A factor that __bandwidth per node within a rack is much higher than bandwidth per node between racks__ makes it valuable to preserve rack locality
  * This can be accomplished by extending Algorithm 2 to give each job two waiting periods

[**[1]**](http://www.cs.berkeley.edu/~matei/papers/2010/eurosys_delay_scheduling.pdf)Delay Scheduling: A Simple Technique for Achieving
Locality and Fairness in Cluster Scheduling
