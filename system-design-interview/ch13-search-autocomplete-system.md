# Chapter 13: Design a Search Autocomplete System

- can also be 'design top k' or 'design top k most searched queries'

Questions to ask:
- Is the matching is supported at the beginning of the query or in the middle too
- How many suggestions should the system return?
- How does the system know which 5 to return?
- Does the system support spell check?
- Are the queries in English?
- Does the system allow capitalization and special characters?
- How many users use the product?

Summary of requirements:
- fast response time
- relevant suggestions
- sorted by popularity or other ranking models
- scalable - handle high traffic volume
- highly available - system should remain available and accessible when part of the system is offline, slows down, or experiences unexpected network errors

Back of the envelope estimations
- 10 million daily active users
- Avg person performs 10 searches a day
- 20 bytes per query string
  - assume we use ASCII character encoding
  - 1 char = 1 byte
  - assume a query contains 4 words, each word contains 5 chars on avg (4 x 5 = 20 bytes per query)
- for every char entered into the search box, a client sends a request to the backend for autocomplete suggestions
- on avg, 20 requests are sent for each search query
  - for ex. once you finish typing 'dinner', there have been 6 requests; 'd', 'di', 'din'...etc.
- ~24,000 QPS = 10mil users * 10 queries/day * 20 chars / 24hrs / 3600 secs
- peak QPS = QPS * 2 ~ 48,000
- assume 20% of daily queries are new
  - 10mil * 10 queries/day * 20bytes/query * 20% = 0.4GB
    - 0.4GB of new data is added to storage daily

High Level Design
- can be broken into two:
Data gathering service
- frequency table that stores the query and the frequency (number of times it has been searched)
- the table is updated to increment the frequency whenever a term is searched

Query service
- we can query the frequency table and filter by the query, sort by the frequence descending and limit the return to 5 records

*This is an acceptable solution if data set is small, but when large, accessing the database becomes a bottleneck

Deep Dive

Trie data structure 
- pronounced 'try'; comes from the word retrieval
- tree like data structure
- root node represents an empty string
- each node stores a character and has 26 children - one for each possible character
- each tree node represents a single word or a prefix string

Algorithm for autocomplete (top k) with trie
- p: length of a prefix(search query)
- n: total # of nodes in a trie
- c: # of children of a given node

1. find the prefix O(p)
2. traverse the subtree from the prefix node to get all valid children; a child is valid if it can form a valid query string O(c)
3. sort the children and get top k O(clogc)

Optimizations

1. Limit the max length of a prefix
- limit p to a small integer number, for ex. 50
- this reduces time complexity to O(1)

2. Cache top search queries at each node
- store top k most frequently used query at each node, for ex 5-10 top queries
- this requires a lot of storage space since we store the top queries at every node
- tradeoff is necessary for fast response time
- this reduces time complexity to O(1)

Data gathering service

- analytics logs
  - stores raw data about search queries
  - append only
  - not indexed

- aggregators
  - aggregate data so it can be easily processed by our system
  - for real-time apps like twitter, we aggregate data in shorter time intervals
  - most use cases, aggregating data less frequently, like once a week should be okay
  - check with interviewer if real-time results are important

- aggregated data
  - table of weekly data
  - stores the query
  - stores the start time of a week
  - stores the frequency of the searched query in that week

- workers
  - set of servers that perform async jobs at regular intervals
  - build the trie data structure and store it in the Trie DB

- trie cache
  - distributed cache system that keep trie in memory for fast read
  - takes a weekly snapshot of DB

- trie DB
  - persistent storage
  - two options:
    - document store: since a new trie is built weekly, we can periodically take a snapshot of it, serialize it, and store it in the DB
      - ex. MongoDB
    - key-value store: a trie can be represented in a hash table form by applying the following logic
      - every prefix in the trie is mapped to a key in a hash table
      - data on each trie node is mapped to a value in a hash table

Query service
1. a search query is sent to the LB
2. the LB routes the request to the API servers
3. API servers get trie from Trie Cache and construct autocomplete suggestions for client
4. in case the data isn't in Trie Cache, we replenish data back to the cache; all subsequent requests for same prefix are returned from the cache; a cache miss can happen when a cache server is out of memory or offline

Optimizations:
- AJAX requests
  - for web apps, browsers usually send AJAX requests to fetch autocomplete results
  - main benefit: sending/receiving a request/response does not refresh the entire page

- browser caching
  - for many apps, the autocomplete search suggestions don't change often
  - autcomplete suggestions can be saved in browser cache to allow subsequent requests to get results from the cache directly


An example:

[Prefixy](https://medium.com/@prefixyteam/how-we-built-prefixy-a-scalable-prefix-search-service-for-powering-autocomplete-c20f98e2eff1)