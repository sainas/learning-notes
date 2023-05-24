# Designing Data-Intensive Applications

[TOC]



## PART I Foundations of Data Systems

### Reliable, Scalable, and Maintainable Applications

Many applications today are data-intensive, as opposed to compute-intensive

- **Reliability**
	
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
	
- **Scalability**


- **Maintainability**