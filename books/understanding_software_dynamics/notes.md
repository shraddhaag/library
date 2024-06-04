# Understanding Software Dynamics

> [!NOTE]
> Explores **optimising transactions that are usually fast but occasionally take much longer**. We assume that always-slow transactions have been identified and fixed.

## Chapter 1 

* Engineers tend to imagine a simple dynamic for their program which generally performs much differently (and slower). 
* A single server in a datacenter runs three kinds of software: 
  1. Programs handling user facing transactions - spare hardware used to handle sudden spikes in traffic. 
  2. Non user facing, batch programs - economically run on idle threats at times of low user facing transactions. 
  3. Supervisory programs - checking server heath, traffic, error rate, etc.  

|Transcational Software|Batch / Offline Software|
|----|-----|
|Important metric is response time.|Important metric is effecient hardware utilisation.|
|Focus on tail latency.|Focus on average latency.|
|even 50% CPU utilisation is bad, as it can affect latency guarentees if load suddenly spikes 3x.|98% CPU utlisation is good.|

* Datacenter software is layered, communicating with each using RPCs synchronously or asnychronously. 
* Each user request and RPC sub-request has a response time goal. 

### Terminology 

* **Transaction / Query / Request** - an input message to a computer system that must be dealt with as a single unit of work.
* **Server** - A computer processing transactions. Each server runs multiple programs where each program likely has multile threads. 
* **Offered Load** - Number of transactions sent per second. 
* **Service** - Collection of programs that handle one particular kind of transaction. Each service has a different offered load and a different latency goal.
* **Execution Skew** - Difference between execution times of parallely executing paths. High execution skews reduced the effectiveness of parallel execution.

### Latency 

* **Transaction Latency / Response time** - Wall-clock time between two events (these two events must be defined precisely). Not constant, probability distribution taken over thousands of transactions per second.
* **Tail Latency** - Slowest transaction in the sample space.
* multiple similar requests to a service will have varying latencies but will cluster around similar values. 

![Latency Histogram](/books/understanding_software_dynamics/latency_histogram.png "A Latency Histogram.")
* peaks - latencies under normal circumstances. 
* long tail - unsual cases that require investigation. 
* Median or maximum latency does not help us understand the graph properly. Percentiles solve this problem. 
* 99 percentile latency - 99% of requests had less or equal repsonse time. 
* reasons for tail latency: 
    * extra work executed - easy to find and fix in test environment using performance tools (which impact performance significantly). 
    * same work, slower execution - hindered transactions, needs to be debugged in production, can only use tools which do not impact performance. 

### Framework to detect performance issues 

Predict - Observe - Compare

* predict: [napkin math](https://github.com/sirupsen/napkin-math), [numbers every programmer should know](https://colin-scott.github.io/personal_website/research/interactive_latency.html)