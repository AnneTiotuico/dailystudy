# Data Systems
### Reliability
- is it continuously working correctly?
- faults can occur at the hardware, software, and the operator (human) level
- redundancy of hardware, well-thoughtout system design (error handling, etc), and testing/monitoring can enhance reliability 

### Scalability
- is it going to work reliably with increased load?
- quantifying load (ex. QPS, read-write ratio, # active users per chat room, hit rate on a cache, etc)
  - twitter's challenge: "fan-out"; many required subsequent requests to serve a single request
    1) with RDBMS
    - a user posts a tweet --> the tweet gets stored in a global collection of tweets
    - a user goes to the timeline --> gets all tweets by all followees, sorted by time
    2) with a cache
    - a user posts a tweet --> the tweet gets stored in a cache of all followers (fan-out)
    - a user goes to the timeline --> gets all tweets from a cache
    3) currently using a hybrid approach
    - fan-out exempted from a user with a large number of followers (the celebrity problem)
  - distribution of followers per user as a key load parameter when discussing scalability
- performance affected by increased load (response time percentiles)
  - latency: duration waiting to be handled
  - response time: time taken from request sent and response received
- add processing capacity to remain reliable under high load

### Maintainability
- how do we make it easier for engineers/operators of the system & avoid creating legacy system?
- operability - monitoring system's health
- simplicity - abstraction into many reusable, simpler components
- evolvability

-------------------------

## Notes
- modern apps are *data-intensive* rather than *compute-intensive*
  - CPU power is rarely the problem
  - the problem usually comes from **managing data**
    - the amount of data
    - complexity of data 
    - speed at which it is changing

### Examples of Data Systems and their application use cases
**Databases**
- stores data so the app or another app can find it again later

**Caches**
- so the app can remember the result of an expensive operation, to speed up reads

**Search Indexes**
- allows users to search data by keyword or filter it in various ways

**Stream Processing**
- send a message to another process, to be handled asynchronously

**Batch Processing**
- periodically crunch a large amount of accumulated data

Using the above building blocks to support your app isn't that simple
- each app has different requirements
- each data system has different characteristics
  - can be hard to combine if a single tool cannot do something alone
- different approaches to caching, search indexes, databases, etc

But the goal is to understand the tradeoffs of each while trying to achieve reliable, scalable and maintainable data systems

### _Thinking About Data Systems
- data storage and processing tools no longer fit into traditional categories
  - there are datastores that are also  used as message queues (Redis)
  - message queues w database-like durability guarantees (Apache Kafka)
- modern apps have demanding/wide-ranging requirements that a single tool no longer meets its storage and data processing needs
  - work is broken down into tasks that can be performed by a single tool and the different tools are stitched together using application code
    - for ex, if you have a cache and index separate from main DB, it's the application code's responsbility to keep the cache and index in sync w main DB

#### **Designing a data system or service**
Questions that come up:
- How do you ensure that the data remains correct and complete, even when things go wrong internally?
- How do you provide consistently good performance to clients, even when parts of your system are degraded?
- How do you scale to handle an increase in load?
- What does a good API for the service look like?

Some factors that influence design:
- skills & experience of people involved
- legacy system dependencies
- timescale for delivery
- your org's tolerance of diff kinds of risk
- regulatory constraints

### _Reliability
- system should continue to work *correctly* (performing the correct function at the desired level of performance) even when things go wrong (hardware or software faults or human error)
- application performs the function that the user expects
- can tolerate the user making mistakes or using the software in unexpected ways
- its performance is good enough for the required use case, under the expected load and data volume
- system prevents any unauthorized access and abuse
- fault-tolerant systems anticipate and can cope with faults/things going wrong

**fault != failure**
- fault: **one component** of the system deviates from its spec
- failure: the **system as a whole** stops provided the required service to the user
- the goal is to design fault-tolerance mechanisms that prevent faults from causing failures

**Testing Tolerance**
- increase rate of faults by triggering them on purpose to make sure your fault-tolerance mechanisms are tested and can correctly handle faults when they occur 
- many critital bugs are due to poor error handling
- Netflix's Chaos Monkey is an example

#### **Hardware Faults**
Usually no/weak correlation between machine failures
  - if one machine's component fails, this doesn't mean another machine's is going to fail too
  - unlikely that several hardware components will fail at the same time

Examples:
- hard disks crash
  - Hard disks have a mean time to failure (MTTF) of about 10 - 50 years
    - on a storage cluster w 10,000 disks, we expect on avg one disk to die per day
- RAM becomes faulty
- power grid has a blackout
- someone unplugs wrong network cable

Solutions:
- add redundancy to the individual hardware components to reduce the failure rate of the system
  - disks may be set up in a RAID configuration
  - server may have dual power supplies and hot-swappable CPUs
  - datacenters may have batteries and diesel generators for backup power
  - doesn't completely prevent hardware problems from causing failures but can often keep a machine running for years
- software fault-tolerance techniques
  - systems can tolerate the loss of entire machines
    - can be patched one node at a time, without downtime of entire system

#### **Software Errors**
Systematic errors within the system are harder to anticipate, and bc they are correlated across nodes, they usually cause many more system failures compared to hardware faults

Examples:
- software bug that causes every instance of an app server to crash when given a particular bad input
  - ex. the leap second on June 30, 2012 that cause many app to hang at the same time due to a bug in the Linux kernel
- runaway process that uses up some shared resource - CPU time, memory, disk space, or network bandwidth
- service that the system depends on that slows down, becomes unresponsive, or starts returning corrupted responses
- cascading failures, where a small fault in one component triggers a fault in another which triggers further faults

Usually triggered by an unusual set of circumstances

Solutions:
- no quick solution, but lots of small things can help
  - think about assumptions & interactions in the system
  - thorough testing
  - process isolation
  - allowing processes to crash & restart
  - measuring, monitoring, and analyzing system behavior in production
  - ex. if a system provides some guarantee like a message queue guarantees that the number of incoming messages equals the outgoing messages, it can constantly check itself while its running and raise an alert if discrepancy is found

#### **Human Errors**
- configuration errors by operators were leading cause of outages vs hardware faults only 10-25%

Solutions:
- design systems that minimize opportunity for error
  - well-designed abstractions, APIs, admin interfaces
- decouple places where most human mistakes are made from places where they can cause failures
  - sandbox environment where people can explore/experiment safely - with real data but without affecting real users
- test thoroughly at all levels - from unit tests to whole-system integration tests and manual tests
  - valuable for covering edge cases
- allow quick & easy recovery from human errors, to minimize the impact in case of failure
  - make it fast to roll back configuration changes
  - roll out new code gradually (unexpected bugs only affect a small subset of users)
  - provide tools to recompute data (in case the old computation was incorrect)
- telemetry: set up detailed & clear monitoring - performance metrics & error rates
  - shows early warning signs
  - allows us to check whether any assumptions or constraints are being violated; helps diagnose the issue when a problem occurs
- implement good management practices & training

*How Important is Reliability?*
- bugs in business apps can cause lost productivity or legal risks
- outages of ecommerce sites can cause lost revenue & damage to reputation
- sometimes we sacrifice reliability in order to 
  - reduce dev cost - when developing a prototype for unproven market
  - reduce operational cost - for a service w a narrow profit margin
- but we should be conscious of when we cut corners


### _Scalability
Even if a system is working reliably today, doesn't meet it still will in the future

- a common reason is increased load
  - ex 10,000 concurrent users to 100,000 concurrent users or 1 mil to 10 mil users
  - processing larger volumes of data than before

**Scalability**: a system's ability to cope w increased load; as the system grows - in data volume, traffic volume, or complexity - there should be reasonable ways of dealing w that growth

Questions to consider:
- If the system grows in a particular way, what are our options for coping w the growth?
- How can we add computing resources to handle the additional load?

#### **Describing Load**
- load parameters: numbers that describe load
  - which operations will be common and which will be rare

Examples:
- requests per second to a web server
- ratio of reads to writes in a DB
- number of simultaneously active users in a chat room
- hit rate on a cache

#### **Describing Performance**
- when you increase a load parameter and keep the system resources (CPU, memory, network bandwidth, etc) unchanged, how is the performance of the system affected?
- when you increase a load parameter, how much do you need to increase the resources if you wan tot keep performance unchanged?

**throughput**: the number of records we can process per second, or the total time it takes to run a job on a dataset of a certain size
  - usually a concern in a batch processing system (for ex. Hadoop)
  - ideally running time of batch job is size of dataset divided by throughput; but running time is usually longer due to skew & needing to wait for the slowest task to complete

**response time**: time between a client sending a request & receiving a response
  - more important metric in online systems over throughput
  - what the client sees; service time: actual time to process the request
  - includes network delays and queueing delays
  - can vary a lot; don't think of it as single number, but a *distribution* of values you can measure

**latency**: duration that a request is waiting to be handled

**Potential causes of slow requests**:
  - process more data
  - latency introduced by a context switch to a background process
  - loss of network packet or TCP transmission
  - garbage colleciton pause
  - page fault forcing a read from disk
  - mechanical vibrations in the server rack

**Measuring response time**
- common to report *average* response time
  - the arithmetic mean: given n values, add up all value and divide by n
  - not a good metric to know your 'typical' response time; doesn't tell you how many users experienced that delay
- better to use *percentiles*
  - take list of response times and sort it from fastest to slowest, then the median is the halfway point
    - better metric to know how long users usually have to wait
    - median is 50th percentile (abbrv: p50)
    - median refers to a single request
- to see how bad outliers are, look at higher percentiles: 95th, 99th and 99.9th
  - these are the response time thresholds at which xx% of requests are faster than that particular threshold
    - for ex, if p95 response time is 1.5 secs, this means 95/100 requests take less than 1.5 secs and 5/100 requests take 1.5 secs or more
  - aka tail latencies
  - important bc these usually indicate important customers with a lot of data
  - queueing delays are a big part of response time at these levels
    - a server can only process a small number of things in parallel
    - only takes a small number of slow requests to hold up processing of subsequent requests (head-of-line blocking)
    - important to measure response times on the client side
- percentiles are often used in service level objectives (SLOs) and service level agreements (SLAs) - contracts that define the expected performance and availability of a service
  - metrics that set expectations for clients of the service & allow them to demand a refund if the SLA is not met
- load testing
  - load generating client needs to send requests independently of the system response time; don't skew measurements
- tail latency amplification: chance of getting a slow call increases if user request requires multiple calls - higher proportion of end-user requests end up being slow

**Approaches for Coping with Load**
- scaling up: vertically scaling, moving to a more powerful machine
- scaling out: horizontally scaling, distributing the load across multiple smaller machines
  - aka shared-nothing architecture
- some systems are *elastic*
  - can automatically add computing resources when they detect a load increase
  - useful if load is unpredictable
- scaled manually
  - a human analyzes the capacity and decides to add more machines to the system
  - simpler and may have fewer operational surprises
- distributing stateless services across multiple machines is straightfoward, but stateful data system from a single node to multiple can be complex
  - keep db on a single node (scale up) until scaling cost or high avail requirements force you to make it distibuted
- no architecture is one-size-fits-all

### _Maintainability
Different people work on the system over time, maintaining current behavior and adapting the system to new use cases and they should be able to work on it productively whether they're in engineering or operations

- majority of the cost of software is not in its initial development, but in its ongoing maintenance
  - fixing bugs
  - keeps its systems operational
  - investigating failures
  - adapting it to new platforms
  - modifying it for new use cases
  - repaying technical debt
  - adding new features

#### **Three design principles for software systems**

**Operabilty**
  - make it easy for new operations teams to keep the system running smoothly

**Simplicity**
  - make it easy for new engineers to understand the system, by removing as much complexity as possible from the system
  - not the same as simplicity of the UI

**Evolvability**
  - make it easy for engineers to make changes to the system in the future
  - adapting it for unanticipated use cases as requirements change
  - aka *extensibility*, *modifiability*, or *plasticity* 

#### **Operability: Making Life Easy for Operations**
- while some aspects of operations can be and should be automated, it's still up to humans to set that up and make sure it works correctly

**Responsibilities of operations teams**
- monitoring the health of the system and quickly restoring service if it goes into a bad state
- tracking down the cause of problems, such as system failures or degraded performance
- keeping software and platforms up to date, including security patches
- keeping tabs on how different systems affect each other, so that a problematic change can be avoided before it causes damage
- anticipating future problems and solving them before they occur (e.g. capacity planning)
- establishing good practices and tools for deployment, configuration management, and more
- performing complex maintenance tasks, such as moving an app from one platform to another
- maintaining the security of the system as configuration changes are made
- defining processes that make operations predictable and help keep the production environment stable
- preserving the org's knowledge about the system, even as individuals come and go

**How data systems can make routine tasks easy**
- providing visibility into the runtime behavior and interals of the system, with good monitoring
- providing good support for automation and integration with standard tools
- avoiding dependency on individual machines 
  - allowing machines to be taken down for maintenance while the system as a whole continues running uninterrupted
- providing good documentation and easy-to-understand operational model
  - "if i do X, Y will happen"
- providing good default behavior, but also giving admin the freedom to override defaults when needed
- self-healing where appropriate, but also giving admin manual control over the system state when needed
- exhibiting predictable behavior, minimizing surprises


#### **Simplicity: Managing Complexity**
- as projects get larger, they become more complex and hard to understand which can slow down productivity of those who work on the system which can increase the cost of maintenance

Symptoms of complexity:
  - explosion of the state space
  - tight coupling of modules
  - tangled dependencies
  - inconsistent naming & terminology
  - hacks aimed at solving performance problems
  - special-casing to work around issues elsewhere

***Abstraction***: a tool that can help remove accidental complexity (not inherent in the problem that the software solved, but from the implementation) 
  - leads to higher-quality software since it benefits all the apps that use the abstracted, reusable component

#### **Evolvability: Making Change Easy**
Likely that your system's requirements will change over time
  - you learn new facts
  - previously unanticipated use cases emerge
  - business priorities change
  - users request new features
  - new platforms replace old ones
  - legal or regulatory requirements change
  - growth of the system forces architectural change

*Agile* working patterns provide a framework for adapting to change
 - technical tools and patterns: test-driven development (TDD) and refactoring

 