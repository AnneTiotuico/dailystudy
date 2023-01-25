# Traditional Hashing
- serverIndex = hash(key) % N, where N is # of servers
- a client (key) contacts a server labeled with the serverIndex calculated above
  - this method is okay, if server size is fixed and data is evenly distributed
  - if server is added or removed, most of keys are redistributed causing cache misses
    - solution: consistent hashing

# Consistent Hashing
- servers are mapped onto a hash ring clockwise
- keys are hashed onto the hash ring, each stored on the closest server clockwise
- adding or removing a server minimally redistributes only affected keys depending on the closest server found clockwise is different
- though key redistribution is minimized, 1) the partitions (hash space between adjacent servers) can become uneven, and 2) key distributions can be non-uniform
  - solution: add virtual nodes

# Consistent Hashing with Virtual Nodes
- each server has its replica virtual nodes
  - ex) no virtual node within a hash ring: s1, s2, s3
  - ex) w/ 2 virtual nodes per server within a hash ring: s1-1, s2-1, s3-1, s1-2, s2-2, s3-2
  - the partition (space between adjacent nodes) becomes more even with more nodes per server
- the goal is to distribute the keys more evenly by using the property of standard deviation that when the sample size gets bigger, the stdev becomes smaller, aka keys are now more evenly distributed
- a tradeoff here is that by introducing virtual nodes, more spaces are needed

# Real-world Examples
- Partitioning component of Amazon's Dynamo DB
- Data partitioning across the cluster in Apache Cassandra
- Discord chat application
