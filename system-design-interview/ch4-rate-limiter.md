# Background
### What is a rate limiter?
- blocks excess calls by limiting the **rate** of incoming requests
- real-life examples:
  - Twitter writes: 300 tws per 3 hrs
  - Google docs reads: 300 per user per 60 secs

### Benefits of using an API rate limiter
- overload prevention
  - DoS attacks, bots, or human misuages 
- cost reduction
  - based on the limit in place, can allocate servers accordingly

# Design a rate limiter
## Prep
#### Clarify scope/assumption
- what kind? client-side or server-side?
  - server-side
- throttle API reqs based on IP, user, or others?
  - should be flexible
- scale of the system?
  - large reqs
- distributed?
  - yes
- a rate limiter = separate service? or within app code?
  - up to you
- inform users about throttling?
  - yes

#### System Requirements
- limit excess requests (drop the excess requests)
- low latency (does not slow down HTTP response time)
- low memory usage
- distributed rate limiting (many services can use a rate limiter)
- exception handling (let users know when being throttled)
- high fault tolerance (its failure doesn't affect the system)

## High-level Design
- client vs. server vs. middleware implemention
  - client implementation is less reliable as it can be forged by malicious actors
  - can implement a rate limiter either directly at the server OR at the middleware (ex. API gateway) that throttles reqs to server

- things to consider:
  - current tech stack of the company; can the current language implement rate limiting on the server-side?
  - what algorithm? if written in server-side, full control over the code, but if using a third-party gateway, options limited
  - current architecture; if already using microservices architecture, include a rate limiter to the API gateway
  - have enough engineering resources to actually build your own rate limiter service? if not, use a third-party API gateway

- algos to consider:
  - token bucket (Amazon, Stripe)
  - leaking bucket
  - fixed window counter
  - sliding window log
  - sliding window counter

- steps:
  - count the requests (by IP, user, or others)
    - if exceeded, throttle
    - else send req to API server
    - count++
  - use Redis, an in-memory cache that allows counter and timeout
    - INCR (increments the counter)
    - EXPIRE (sets a timeout for the counter)

1. client sends a request to the middleware
2. the middleware fetches the counter from Redis
3. the middleware checks if limit reached
  - if so, req rejected; sends an 429 error response
  - else, req sent to the server, counter incremented, then saved to Redis
