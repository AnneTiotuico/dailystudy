# Prep
### Clarify the Requirements for Building a Notification System
- type? push notification, SMS message, and email
- real-time? "soft" real-time; a slight delay acceptable, meaning users get the notification as soon as possible
- supported devices? iOS & android devices, laptop/desktop
- what triggers notification? by the client or scheduled on server
- opt-out option? yes
- how many notifications per day? 10M mobile, 1M SMS msgs, and 5M emails per day

# High-level Design
What are the high-level components of a scalable notification system that supports push mobile push notification (iOS & Android), SMS message, and email?
- identify the type of the notification to select a third party service
- gather the contact info to use as an endpoint for the notification
- architect an event-driven architecture for the triggering, sending, receiving flow
- then consider optimization methods with scaling in mind

### Types
Mobile Push Notification - iOS or Android, SMS Message, Email
- Provider (builds/sends notification requests) --> 3rd party service (depending on the type) --> client

### Contact Info Gathering
Mobile device tokens, Phone numbers, Email addresses
- when a user signs up, API server collects and stores contact info to the database
- ex) user table has user_id, email, phone_number, etc; device table has device_token, user_id, etc
  - a user can have multiple devices (1:M)

### Notification Sending/Receiving
Flow
1) Services: triggers notification sending event 2) Notification System: provides APIs to the services, builds notification payloads for 3rd party services
3) 3rd Party Services: delivers notifications
  - consider extensibility (aka easy to plug/unplug third party service) & regions/markets it supports
4) Users/Devices: receives notifications on their device

Problems of a Single Notification Server (not multiple)
1) SPOF
2) hard to scale db, caches, and different processing components within the system independently
3) performance bottleneck as processing/sending notifications can be resource intensive

### Optimization
1) add more notification servers & set up automatic horizontal scaling
2) decouple the database and cache from the notification server
3) add message queues to allow for parallel processing (serves as buffers when dealing with a high volume)
  - add workers that pull notification events from the message queues and send to third-party services

### Recap
1) a service calls APIs (provided by notification servers) to send notifications
2) notification servers fetch metadata (user info, device token, notification settings) from the cache or db
3) a notification event is sent to the corresponding queue for processing
4) workers pull notification events from message queues
5) workers send notifications to third party services
6) third party services send notifications to user devices

# Deep-dive
### Preventing Data Loss 
- include a notification log for data persistence
- for duplicates, a notification that has a "seen" event_id will be discarded

### Additional Components
- notification template for consistent formatting and saving time
- notification setting for users to opt in or out
  - notification setting table with user_id, channel, opt_in fields
- rate limit the notifications a user can receive
- retry mechanism to add a notification to a queue if failed
- security service to only allow verified clients to send push notifications using the notification system API
- monitor queued notifications; if number high, then worker not processing fast enough; adjust # of workers as needed
- analytics service for event tracking to collect/analyze customer behavior data

### Recap
1) notification servers now have authentication (security) and rate-limiting (respect user) features
2) there's a retry on error mechanism for failed notifications put back in the messaging queue (reliability)
3) workers use a notification template and persist data by storing notifications into the notification log
4) monitoring/tracking systems are added for system health checks and analysis

# Other Resources
- https://www.notificationapi.com/blog/notification-service-design-with-architectural-diagrams
