# Background
### What is a web crawler?
- known as a robot or spider, used by search engines to discover new/updated content on the web
- it collects a few web pages, follows links on those pages to collect new content

### What is it used for?
- search engine indexing: collects web pages, create a local index for SE; ex) Google bot for Google search engine
- web archiving: collect, preserve for future use; ex) US Library of Congress
- web mining: ex) download shareholder meetings for fin firms
- web monitoring: ex) copyright/trademark infringement 

# Requiements
### Basic Algorithm
- given a set of URLs..
  - download all web pages using the URLs
  - extract URLs from these websites
    - all new URLs to the set
  - continue til no more URLs?

### Clarify Requiements
- main purpose? search engine indexing
- scale -- how many web pages collected / month? 1B pages
- content type? HTML only
- consider newly added or edited web pages
- storage length for collected HTML? up to 5 years
- duplicate content handling? ignore duplicates

### Characteristics of a Good Web Crawler
- scalable (use parallelization for efficiency), robust (handle edge cases), polite (limit rate on your end!), and extensible (requires minimal changes for new support)

### Quantify Assumptions
- QPS (1B web pages per month)
  - avg: 1^9 / 30 days / 18600 sec = ~400 pgs per sec
  - peak: 2 * QPS = 800 pgs per sec
- Storage (500 kb per one web pg; 5 year storage)
  - per month: 1^9 * 500 kb = 500 tb
  - per 5 years: 500 tb * 12 mo * 5 yr = 30 pb

# High-level Design
1. determine a list of known URLs as a starting point depending on scope and strategy of crawling
2. URL Frontier has a queue with URLs to be visited
3. HTML Downloader gets IP from DNS that resolves the URL given
4. HTML Downloader downloads the HTML of the URL
5. Content Parser parses the HTML content to verify whether the content is not malformed
6. Content Seen checks the Content DB whether the content already exists
7. If not, Link Extractor retrieves all links from the content and turn it into an absolute URL
8. Extracted and formated URLs are sent to the URL Filter
8. URL Filter filters out any error or blacklisted links
9. URL Seen checks if the found URL already exists in the URL db or in the URL Frontier
10. if not, add the URL to the Frontier

# Deep-dive
### Depth-first vs Breadth-first
- think of the web as a directed graph (pages = nodes; URLs = edges)
- since it can get very deep, BFS with a queue a common algo
  - cons
    - downloading web pages in parallel (when they are from the same host), the server gets flooded w/ requests, which is considered "impolite" and can be treated as DoS attack
    - standard BFS doesn't have "priority" of a URL

### URL Frontier
- a data structure that stores "to-be-downloaded" URLs
- this is where politeness & prioritize URLs

URL Prioritization (which URL is more important?)
- a "prioritizer" computes the priorities based on a URL's usefulness measured by PageRank, website traffic, update frequency, etc
- the input URLs are organized into multiple queues (FIFO) with each associated with a specific priority
- a queue selector chooses a queue with a bias towards high priority queue

Politeness (download one page at a time from the same host)
- a mapping table contains hostname-to-worker pairs
- a queue router ensures each queue only contains URLs from the same host
  - mutliple queues (FIFO) with each associated with a unique host
- a queue selector has the queue selection logic for each worker
  - multiple worker threads (download one web page at a time with delays in between)

Frontier Design Overview w/ Prioritization & Politeness
- organize the input URLs into priority queues, then organizhe them into politeness queues based on the mapping table, which then each worker is selected to process the URL, with delays in between

Other Details Regarding Frontier
- Freshness: recrawl based on web pages' update history or recrawl important pages first/more frequently
- Storage: store in disk since there's a lot of URLs, and maintain buffers in memory for enqueue/dequeue operations and have the data periodically written to the disk

### HTML Downloader
Robots.txt
- Robots.txt (or Robots Exclusion Protocol) is a standard used by websites to communicate with crawlers regarding what pages can be downloaded; a crawler must check this and follow the rules!

Performance
- for high performance...
  - crawl jobs can be distributed into multiple servers (each running multiple threads) geographically (to reduce latency)- can cache DNS resolver (cron job to update the map)
  - set short timeouts for unresponsive web servers

Robustness
- implement consistent hashing to help distribute loads among downloaders
- save crawl states/data so it's recoverable when lost
- handle exceptions
- validate data

Extensibility
- make the system flexible for when it has to change to support new content types
  - make it easy to plug in new modules for new logic

Filter Content
- filter out duplicates, no-val content, and spider traps that causes an infinite loop
  - solution: set a maximal length for URLs
