# Background
- `auto_increment` attribute in a traditional db doesn't scale well since generating unique IDs across multiple dbs with minimal delay in a distributed environment is challenging

# Clarify Requirements
- characteristics of unique IDs? unique & sortable
- increment by 1? not necessarily; increments by time (id created later is larger)
- only numerical values? yes
- length? 64-bit
- what is the scale of the system? be able to generate 10k IDs per sec

# High-level Design
- what are the options for generating unique IDs in distributed systems?
  - multi-master replication
  - UUID (universally unique identifier)
  - ticket server
  - twitter snowflake approach

### Multi-master Replication
- it uses auto_increment feature, in which each id for a db increments by k, where k is the number of nodes in use
- drawbacks
  - hard to scale with multiple data centers
  - IDs do not go up with time across multiple servers
  - doesn't scale well when a server added/removed

### UUID
- it's a 128-bit number used to identify info in computer systems and has a very low probability of getting duplicates (ex. `09c93e62-50b4-468d-bf8a-c07e1040bfb2`)
- pros
  - simple, as no coordination needed between servers, and thus easy to scale (each server independently generate IDs)
- cons (related to the requirements given)
  - each id 128-bit long (doesn't meet the requirement that is 64-bit)
  - ids don't go up with time
  - ids not numeric

### Ticket Server
- Flicker developed ticket servers to generate distributed primary keys (web servers talk to one ticket server, a centralized auto_increment server)
- pros
  - numeric IDs and easy to implement (great for small to medium scale apps)
- cons
  - one ticket server = SPOF
    - solution is to add more ticket servers, but it comes with synchronization challenges

### Twitter's "Snowflake" Approach
- divide and conquer; each id has sections that represent different information
  - sign bit (1 bit) + timestamp (41 bits) + datacenter id (5 bits) + machine id (5 bits) + sequence number (12 bits)

# Deep-dive - Snowflake
- snowflake approach supports scalability in a distributed environment
- datacenter id & machine id are chosen at the system's startup time and generally fixed (modifying can lead to id conflicts)
- timestamp & sequence numbers generated when the id generator is running

### Timestamp
- sortable by time
- binary representation of UTC
- max 41 bits; id generator will work for 2 ^ 41 - 1 ms or ~69 yrs

### Sequence Number
- 12 bits or 2 ^ 12 combos; a machine can support a maximum of 2 ^ 12 (or 4096) new ids per ms
- this is 0 unless 1+ id generated in a ms on the same server

# Additional Consideration
1) clock synchronization for when a server is running on multiple cores and not have the same clock
  - solution: Network Time Protocol
2) section length tuning
  - fewer seq number + more timestamp bits = more effective for low concurrency and long-term applications
3) high availability of an id generator

# FYI (OpenAI answer)
a "star" schema and a "snowflake" schema
- a "star" schema dimension tables are denormalized and in a "snowflake" schema dimension tables are normalized.
