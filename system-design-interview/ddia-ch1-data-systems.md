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
