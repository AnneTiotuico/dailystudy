# Establish Design Scope
- important features? uploading a video, whatching a video
- supported clients? mobile apps, web browsers, smart TV
- how many daily active users? 5M
- avg daily time spent? 30 mins
- international user support? yes, a larget % of users are international users
- supported video resolutions? most video resolutions and formats are supported
- encryption? required
- file size requirement for vids? max vid size is 1GB
- leverage existing cloud infrastructures? yes (uncommon to build everything from scratch)

### Requirements
- design a video streaming service w/ fast uploads, smooth video streaming, support for different vid qualities, high availability/scalability/reliability for mobile, web, and smart TV app

### Estimation - Stroage for Writes
- 5M DAU
- watch 5 vids / day
- 10% of users upload 1 vid / day
- avg vid size is 300MB
==> total daily storage space: 5M * 10% * 300MB = 150TB (use a blob storage service for storing source vids)

### Estimation - CDN Cost for Reads
- assume we are using AWS CloudFront for CDN service
  - charged for data transferred out of the CDN
- 100% traffic served from the US
- avg cost per GB is $0.02
==> 5M users * 5 videos * 300MB * $0.02 = $150k per day (expensive!!!)

# High-level Design
- Client ---> CDN for streaming vid
- Client ---> API servers for everything else (feed recommendation, generating vid upload URL, updating metadata db/cache, user signup, etc)

### Video Uploading Flow
- client uploads original vids
  - stored in a blob storage system
  - transcoded by transcoding servers, meaning a video format is converted to another format for best video streaming for different devices/bandwidth capabilities
- the original vids are transcoded
  - stored in a blob storage system
  - cached in CDN and streamed from CDN when user clicks play
- after transcoding...
  - when transcoding event completes, info pushed to a message queue
  - workers pull event from the queue and update the video metadata cache and db
    - cached for better performance
    - db sharded/replicated for high availabilit

### Video Streaming Flow
- videos are streamed directly from CDN -- the edge server closest to you will deliver the video aka very low latency

# Deep Dive
### Video Transcoding
Background
- what's the purpose of transcoding a video?
  - for a video to be played smoothly on a device, it must be encoded to compatible bitrates & formats (higher bitrate = higher quality)
- what's the point of having multiple qualities of a video?
  - network conditions vary; for a smoother user experience, switching the vid quality automatically is sometimes essential
- most encoding formats contain two parts -- container (ex. .avi, .mov, .mp4) and codecs (compression algo to reduce vid size; ex. H.264, VP9, HEVC)
- DAG (directed acyclic graph) model for transcoding (similar to FB's transcoding system to achieve parallelism)
  - stage 1: original video broken into video, audio, metadata
  - stage 2:
    - video broken into tasks including video encoding, thumbnail, watermark, etc
    - audio encoded
  - post video tasks & audio encoding, they are assembled back

The Video Transcoding Architecture
- components: preprocessor, DAG scheduler, resource manager, task workers, temp storage, and encoded video as the output
  - preprocessor splits video, generates DAG config files, and cache data in a temporary storage for retry in case of vid encoding failure
  - DAG scheduler splits a DAG graph into two stages (as described above)
  - resource manager manages the efficiency of resource allocation using queues and scheduler
  - task workers run tasks defined in DAG
  - temp storage -- many options depending on what you are storing
    - metadata: frequently accessed by workers & data size is small --> in-memory
    - video/audio data: blob storage like S3
    - data in temp storage is freed up when the corresponding video processing is complete

### System Optimizations
- goal: speed, safety, cost-saving
- optimize uploading speed
  - split a vid into smaller chunks and upload many in parallel
  - use CDN as upload center so it's placed close to the uploaders
  - modify the system to be loosely coupled to allow for parallelism in the system
- safety
  - use pre-signed URL so only authorized users upload vids to the right location
  - protect copyrighted videos by using Digital Rights Management systems, AES encryption, or visual watermarking
- cost-saving
  - only server the most popular vids from CDN since CDN is expensive
  - for less popular content, don't need many encoded versions
  - short vids can be encoded on-demand
  - know in what region the content is popular; only store the content in that region's CDN

### Error Handling
- error will happen, so build a highly fault-tolerant system, meaning handling errors gracefully & recover FAST.
  - recoverable error due to video segment failure to transcode --> retry a few times (before calling it nont-recoverable)
  - non-recoverable error due to malformed video format --> return a proper error code to the client
- in general, retry or use a replica (since system built to be highly available)
