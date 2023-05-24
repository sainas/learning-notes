# Designing Data-Intensive Applications

[TOC]



## PART I. Foundations of Data Systems

### Chapter 1. Reliable, Scalable, and Maintainable Applications

Many applications today are data-intensive, as opposed to compute-intensive

#### Reliability

Fault and failure: System need to be fault-tolerance

- Hardware Errors

  1. hard disks fails -> add redundancy (reduce rate) -> larger data requires larger number of machine (increase rate)  -> more virtual machines (increase rate)
  2. Such tolerance also allows rolling upgrade (patch one node a time, no downtime)

- Software Errors

  - More correlated across nodes

  - Often because it made assumptions and that assumptions was no longer true

  Solutions

  - (Before launch) Testing, in design, isolate process and allows crash
  - (After launch) monitoring, analyzing, alerting

- Human Errors

  - Config errors by human is the leading cause of outages

  Solutions

  - System: well-design abstraction
  - Decouple the place of mistakes: use sandbox
  - Test
  - Quick and easy recovery. Allows recomputing data
  - Monitoring
  - Management and training

#### Scalability

  It's not a label: "X is scalable" or "Y doesn't scale". But rather "If the system grows in a particular way, what are the options"
  **Describing Load**

  - *Load parameter*
    - e.g. requests per second, ration of reads to writes, hit rate on cache

  - Twitter example

    Approach 1: user's timeline from query
    Approach 2: cache for each user and inserting on new tweet
    Twitter moved from 1 to 2, then celebrities with many followers got slow.
    Solution: celebrities use Approach 1

    In this example, the distribution of followers per user is a key load parameter for scalability

  **Describing Performance**

- batch processing - throughput; service - response time

- Latency vs response time: 
  response time = queueing + processing + network delay 
  Latency = network delay (maybe plus queueing?)

- Average isn't good enough, median or percentiles is better. the 95th, 99th, and 99.9th percentiles are common (abbreviated p95, p99, and p999).

- Tail latencies are important
  e.g. Amazon use 99.9th percentile, since slowest requests are often with the most data and are often most valuable customers
- How to measure
  - Measure response time on client side
    A few slow requests can delay the whole queue, even those subsequent request are fast *"head-of-line blocking"*
  - Keep sending request, rather than wait for response and then send
  - *"tail latency amplification"*: one user requests takes several backend requests, the slowest backend request slows down the entire user request
  - rolling window metrics: many ways to calculate a good approximation: forward decay, t-digest, or HdrHistogram. Besides, for multiple machine, avg percentile doesn't make sense, need to aggregate histograms.

**Approaches for Coping with Load**

vertical scaling (more powerful machine) 
horizontal scaling (multiple machines, aka *shared-nothing* architecture)

Elastic: automatic
scales manually: less surprise

Advices:

- Stateful data is harder to distribute. Single node first until not viable

- It is conceivable that distributed data systems will become the default in the future, even for use cases that don’t handle large volumes of data or traffic. But not yet!

- The architecture of systems that operate at large scale is usually highly specific to the application. No one-size-fits-all scalable architecture *"magic scaling sauce"*

  volume of reads / writes / data to store / complexity of the data / response time / access pattern

- Depends on load parameter assumption. If they are wrong, wasted or counterproductive
  Early stage, future load are still hypothetical, product feature quick iteration is usually more important

#### Maintainability

**Operability**: Making Life Easy for Operations

Anticipating future, better definition and knowledge sharing.
Visibility into runtime and system. Good default as well as admin freedom

**Simplicity**: Managing Complexity

One of the best tools we have for removing *accidental complexity* is *abstraction.*

**Evolvability**: Making Change Easy
Agile, test-driven development