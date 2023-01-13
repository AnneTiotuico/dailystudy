# Chapter 12: Design a Chat System

- chat apps have different functions for different people
- clarify the exact requirements before moving forward
  - group chat focus vs one-on-one chat
    - group chat examples: office chat apps like Slack, game chat apps like Discord
      - focus on large group interaction and low voice chat latency
    - one-on-one examples: Facebook messenger, WeChat, WhatsApp
  - explore feature requirements

## Step 1: Understand the problem and establish design scope

<details>
<summary>What kind of chat app, one-on-one or group based?</summary>

  - Both one-on-one and group chat
</details>

<details>
<summary>Will this be a mobile app, a web app or both?
</summary>

  - Both
</details>

<details>
<summary>What is the scale - startup or massive scale?
</summary>

  - It should support 50 million daily active users (DAU)
</details>

<details>
<summary>For group chat, what is the member limit?
</summary>

  - A max of 100 people
</details>

<details>
<summary>Which features are important? Can it support attachments?
</summary>

  - one-on-one chat, group chat, online indicator
  - the system only supports text messages
</details>

<details>
<summary>Is there a message size limit?
</summary>

  - Yes, text length should be less than 100,000 characters long
</details>

<details>
<summary>Is end-to-end encryption required?
</summary>

  - Not required for now, but we'll discuss if time allows
</details>

<details>
<summary>How long should the chat history be stored?
</summary>

  - Forever
</details>
<br>

> Note: the design will focus on a chat app like Facebook messenger that emphasizes the following features:
>  - a one-on-one chat with low delivery latency
>  - a small group chat (100 people max)
>  - online presence
>  - multiple device support; same account can be logged into multiple accounts at the same time
>  - push notifications

## Step 2: Propose high-level design and get input
- understand how clients and servers communicate
- in a chat system, clients can be:
  - mobile apps
  - web apps
- clients don't directly communicate w each other
  - they connect to a chat service 

<details>
<summary>Fundamental operations
</summary>

- receive messages from other clients
- find the right recipients for each message & relay the message to the recipients
- if a recipient is offline, hold the messages for that recipient on the server until she is online
</details>
<br>

### **Network protocols**
- choice of network protocols is important for a chat service
- when the sender sends a message to the chat service, it uses HTTP (most common web protocol) which includes the `keep-alive` header
  - `keep-alive` is efficient because it allows a client to maintain a persistent connection with the chat service
    - also reduces the number of TCP handshakes
- receiver side is more complicated since HTTP is client-initiated and it's more complicated to send messages from the server
  - techniques used to simulate a server-initiated connection
    - polling
    - long polling
    - WebSocket

### Polling
- client periodically asks server if there are messages available
- could be costly, depending on polling frequency
  - consumes server resources even when often times there aren't messages available

### Long polling
- client holds the connection open until there are new messages available or a timeout threshold has been reached
- once client receives new messages, it immediately sends another request to the server, restarting the process

Drawbacks:
- sender & receiver may not connect to the same chat server
  - HTTP based servers are usually stateless
  - if using round robin for load balancing, server that receives the message might not have a long-polling connection w the client who recieves the message
- server has no good way to tell if client is disconnected
- inefficient; if a user doesn't chat much, long polling still makes periodic connections after timeouts

### WebSocket
- most common solution for sending asynchronous updates from server to client
- client-initiated
- **bi-directional**
  - can be used for both sender and receiver sides 
    - simplifies the design and makes implementation on both sides easier
- persistent
  - efficient connection management is critical on the server-side
- starts as a HTTP connection and could be 'upgraded' via some well-defined handshake to a WebSocket connection
- server could send updates to client via the WebSocket connection
- generally work even if a firewall is in place
  - because they use port 80 or 443 which are also used by HTTP/HTTPS

### **High-level design**
- WebSocket is used for main protocol between client and server
  - but everything else like (sign up, login, user profile, etc) can use traditional request/response method over HTTP

Three major categories:
- stateless services
- stateful services
- third-party integration

**Stateless Services**
- traditional public-facing request/response services
  - used to manage login, signup, user profile, etc
  - most common features of websites and apps
- sit behind a load balancer which routes requests to the correct services based on the request paths
- can be monolithic of individual microservices
- don't need to build many of these stateless services ourselves since there are services that can be integrated easily
- service discovery will be the focus
  - primary job is to give client a list of DNS host names of chat servers that the client could connect to

**Stateful Services**
- only stateful service is the chat service
- each client maintains a persistent network connection to a chat server
  - client normally doesn't switch to another chat server as long as the server is still available
- service discovery coordinates closely w the chat service to avoid overloading

**Third-party integration**
- for a chat app, push notification is important
- a way to inform users when new messages have arrived, even when app isn't running

**Scalability**
- on a small scale, all services above could fit on one server
- the number of concurrent connections that a server can handle will most likely be the limiting factor
- assumptions:
  - 1 mil concurrent users
  - 10k of memory on the server for each user connection
    - rough number; very dependent on language choice
  - 10GB of memory needed to hold all connections on one machine
- **DON'T** propose a design where everything fits on a single server
  - red flag to interviewers
  - risk of single point of failure (SPOF)

**Adjusted high level design**
- chat servers faciliate message sending/receiving
- presence servers manage online/offline status
- API servers handle everything including user login, signup, change profile, etc
- notification servers send push notifications
- key-value store is used to store chat history
  - when an offline user comes online, she will see all previous chat history

**Storage**

Two types of data in a typical chat system
- generic data
  - user profile
  - settings
  - user friends list
  - stored in robust & reliable relational DBs
  - replication and sharding are common techniques to satisfy availability and scalability requirements
- chat history
  - huge amount of data (ex. 60 bil message/day)
  - only recent chats are accessed frequently
    - but users may use features that require random access of data
      - search
      - view mentions
      - jump to specific messages
      - these could be supported by data access layer
  - read/write ratio is about 1:1 for one-on-one chat apps

Kev-value stores recommended because:
- allow easy horizontal scaling
- provide very low latency to access data
- relational databases do not handle long tail of data well
  - when indexes grow, random access is expensive
- used by other proven reliable chat apps
  - FB messenger uses HBase; Discord uses Cassandra

**Data models**
- message data is the most important data in chat apps

Message table for 1on1 chat
- primary key: `message_id` (bigint)
  - helps decide message sequence
- `message_from` (bigint)
- `message_to` (bigint)
- `content` (text)
- `created_at` (timestamp)
  - can't be used for message sequence since messages can be sent at the exact same time

Message table for group chat
- composite primary key: `channel_id` (bigint), `message_id` (bigint)
  - `channel_id` is the partition key since all queries in a group chat operate in a channel
- `user_id` (bigint)
- `content` (text)
- `created_at` (timestamp)

**Message ID**
- must satisfy two requirements
  - should be unique
  - should be sortable by time
    - new rows have higher IDs than old ones

Solutions
- 'auto_increment' keyword in MySQL
  - NoSQL DBs don't usually provide this feature
- global 64-bit sequence number generator like Snowflake
- local sequence number generator
  - IDs are only unique within a group
  - maintaining message sequence w/in 1on1 chanell or group channel is sufficient
  - easier to implement compared to global ID

## Step 3: Design deep dive
- usually expected to deep dive into a few components from high-level design
- for chat system: 
  - service discovery
  - messaging flows
  - online/offline indicators

### **Service discovery**
- primary role is to recommend best chat server for a client based on criteria (geographical location, server capacity, etc)
- Apache Zookeeper: popular open source solution
  - registers all available chat servers and picks the best for a client based on criteria
  - example flow:
    - user A tries to log into app
    - load balancer sends login request to API servers
    - after backend authenticates user, service discovery finds the best chat server for user A
      - server 2 is chosen and its info is returned to user A
    - user A connects to chat server 2 through WS

### **Message flows**

**1 on 1 chat flow**

1\. user A sends chat message to chat server 1

2\. chat server 1 obtains a message ID from the ID generator

3\. chat server 1 sends the message to the message sync queue

4\. the message is stored in a key-value store

5.a. if user B is online, the message is forwarded to chat server 2 where user B is connected

5.b. if user B is offline, a push notification is sent from push notification servers

6\. chat server 2 fowards the message to user B. There is a persistent WebSocket connnection between user B and chat server 2

**Message synchronization across multiple devices**
- user A has two devices: phone and laptop
  - when user A logs in to the chat app w her phone, it establishes a WebSocket connection w chat server 1
  - there is also a connection between the laptop and chat server 1
- each device maintains a variable `cur_max_message_id` 
  - keeps track of the latest message ID on the device
- messages are consdered as new messages if:
  - the recepient ID is equal to the currently logged in user iD
  - message ID in the key-value store is larger than `cur_max_message_id`
- with unique `cur_max_message_id` on each device, message synchonization is easy since each device can get new messages from the KV store

**Small group chat flow**

3 group chat members
- each has their own message sync queue

1 user sends a message:
- user A sends message in a group chat
- the message is copied to each group member's message sync queue (one for user B, and one for user C)
- message queue is like an inbox for a recipient
- good design for small group chat because:
  - simplifies message sync flow since each client only needs to check its own queue to get new messages
  - when a group is small, storing a copy in each recipient's queue isn't too expensive

Recipient receives messages from multiple users:
- the message sync queue holds all the messages from different senders

### **Online presence**
- essential feature for chat apps
- in the high-level design, presence servers are responsible for managing online status and communicating w clients via WebSocket

Flows that trigger online status change:

**User login**
- after a WS connection is built between the client and real-time service, user A's online status and `last_active_at` timestamp are saved in the KV store
- indicator shows user is online after she logs in

**User logout**
- user logs out and sends request to API servers which goes to presence servers and updates the online status to offline in the KV store
- indicator shows user is offline

**User disconnection**
- internet connection is inconsistent and unreliable so we must handle this issue
- when user disconnects, the persistent connection between client and server is lost
- naive way:
  - change user status to offline
  - change back to online when they reconnect
  - flaw:
    - users may connect/disconnect to the internet frequently in a short time
    - updating the presence indicator too often results in poor user experience

- heartbeat mechanism:
  - periodically, an online client sends a heartbeat event to presence servers
  - if server receive a heartbeat event w/in certain amount of time from client, user is considered online
  - for ex. client sends heartbeat ever 5 secs, after sending 3 heartbeats the client is disconnected and doesn't reconnect within 30 secs, the online status is changed to offline
  
  **Online status fanout**
  - to let friend's know a user's status changes, presence servers use a publish-subscribe model
    - each friend pair maintains a channel
    - when user A's status changes, it publishes the event to three channels, A-B, A-C and A-D
      - these channels are subscribed by user B, C and D respectively
  - this model works for small groups, but as the member group becomes larger, a better solution is to get online status only when a user enters a group or manually refreshes the friend list

  ## Step 4: Wrap up
  
  Additional talking points:
  - extend chat app to support medial files (photos, videos)
    - media files are significantly larger than text in size
    - compression, cloud storage, and thumbnails are topics to discuss
  - end-to-end encryption
    - only the sender and recipient can read messages
  - caching messages on the client-side 
    - reduce the data transfer between client and server
  - improve load time
    - for ex. slack built a geographically distributed network to cache user' data, channels, etc
  - error handling
    - chat server error
      - if chat server goes offline, service discovery will provide a new chat server for clients to restablish connections with
    - message resent mechanism
      - retry and queueing are common techniques
    