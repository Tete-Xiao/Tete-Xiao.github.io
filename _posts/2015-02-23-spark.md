---
layout: post
title: "Spark : Resilient Distributed Dataset for efficient in memory computing"
date: 2015-02-23
comments: True
categories: "big-data"
description: "This article is basically a high level summary for a 3 minutes presentation that lead the audience to have a intuition about what Spark is and how it achieve efficient and fault-tolerant in-memory computing."
---

This article is basically a high level summary for a 3 minutes presentation that lead the audience to have a intuition about what Spark is and how it achieve efficient and fault-tolerant in-memory computing.

<!--more-->

<hr class="soft"/>

### Intro

UC Berkley proposed a fault tolerant abstraction called Resilient Distributed Datasets for in memory cluster computing. And Spark System is their implementation for this proposal. More resources about this inspiring article could be found [*here*](https://www.usenix.org/conference/nsdi12/technical-sessions/presentation/zaharia).

### Problem & Motivation
* Problem:
  * As big data analysis becomes more and more popular nowadays, the demand for some particular iterative algorithms (Page Rank, Logistic Regression ...) to have efficient computing framework for data sharing and reusing exploits.  
* Motivation:
  * Keep intermediate data structure persistent in memory so as to reuse it along the way of cluster computing procedure

### Design Decision
* Key Features
1. Data sharing efficiency
2. Fault-tolerance
* Solution
  * Use a **Restricted form of distributed shared memory**
    * Immutable, partitioned collections of records and spreads them across the nodes in the computing clusters
    * Read-only after the creation of RDD, so as to maintain good read concurrency property

  * RDDs can only be created/“written” through **coarse-grained deterministic transformations** on either data in stable storage (e.g. disk) or other RDDs
    * Restrict application to perform bulk write => Sacrifice for efficient fault tolerance
        * Bulk can be scheduled based on data locality => improve the performance
        * It makes RDD degrade well with Insufficient memory => sequential read from disk

  * Use **Ef ficient fault recovery mechanism** with ***lineage graph***
    * Log *logical transformations* that RDDs applied, so as to reduce the overhead of heavy logs
    * When there is some node failures => Just re-compute lost partitions on failure using its parent

  * Provide Partition control for user
    * Let user decide the persistence level (on disk, on memory), so they can make right decision for their specific application

  * Another contribution besides is the **Generality of Spark**
    * Can express many existing parallel model (Those naturally apply the same operation to many items such as MapReduce, Iterative MapReduce, DryadLING, Pregel, SQL)
    * Enables app to efficiently intermix these models
### Performance Benchmark
* Data mining (such as Page Rank) tasks: 40x faster than Hadoop
* Machine learning (such as Logistic Regression) tasks: 20x faster than Hadoop

### Conclusion
* RDD offer a simple and efficient programming model for a broad range of applications
* Leverage the coarse-grained nature of many parallel algorithms for low-overhead recovery

### Reference
[ **[1]** ](https://www.cs.berkeley.edu/~matei/papers/2012/nsdi_spark.pdf) Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster Computing
