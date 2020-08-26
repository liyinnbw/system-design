# System Design Considerations

## Understand Requirements
- Functional (What the system can do)
- Non-functional
  - Availability
  - Speed
  - Security
- Additional (non-standard requirements)

## System Capacity Estimation
- Traffic (Bandwidth)
  - reads x (datasize_per_read) /s
  - writes x (datasize_per_write) /s
- Storage
  - write_traffic x storage_duration
- Memory (Cache)
  - 80-20 rule => daily_read_traffic x 0.2
  
## Define API
- Define basic API based on functional requirement
- You may slowly enrich this API to satisfy non-functional requirements

## Design Database
- SQL or NoSQL (Not only SQL)?

|Factor|SQL|NoSQL|
|---|---|---|
|Scalability|Easy to horizontally scale|Hard to horizontally scale due to [CAP](https://en.wikipedia.org/wiki/CAP_theorem) theorem, hard to remain ACID, expensive to do cross-server query|
|Cache friendly|May not be|Yes, since cache is NoSQL key-value pair, so cache & DB can adopt similar structure, making it easier to cache|
|Data Structure|Unstructured data (can be multimedia)|Structured (only text data with fixed type/size)|
|Schema flexibility|Schema should be determined beforehand, hard to change later|Schema can be easily changed later|
|ACID (atomicity, consistency, isolation, durability) compliance|Usually only AID, with eventual consistency| ACID (usecase: money transfer)|
|Data analysis|Easy due to structured data & SQL language|Harder, due to unstructured data| 
- Schema


## Design for Functional Requirements
- Draw UML or simple workflow diagram to help understanding
  - User request, App server, DB, arrow to represent data flow
- Explore various design options, state their pros and cons before making a decision

## Design for Non-functional Requirements
- Availability & Reliability
  - Replicate
    - Load balancer
      - Use a pair (HA peers)
      - Each keep track of the other's status periodically (heartbeat)
      - If one fails, the other takes over in seconds (either through on-premise floating IP mechanism, or a cloud TCP/UDP router [read more](https://cloud.google.com/solutions/best-practices-floating-ip-addresses))
    - Server (behind load balancer)
    - DB (master (read-write) & slave (read only))
  - Separate Read/Write to different servers
    - Each server can handle limited connections at any single time
    - If many time-consuming writing process (such as upload) take place at the same time, read requests cannot be handled
- Scale out (horizontal scaling: add servers, vertical scaling: add CPU, RAM, Dist etc. in a single server)
  - Replicate servers (for both availability & handle more request)
    - App servers
    - DB servers
    - Cache servers
  - Load balancing
    - Location
      - Client->LB->App
      - App->LB->DB
      - App->LB->Cache
      - Cache->LB->DB
    - Balancing strategy
      - Round robin
      - load-dependent
    - Note
      - load-balancer can be single point of failure, always has a duplicate idle load-balancer
      - load-balancer itself has limited resources to handle incoming connections/requests (but is much larger than app server)
      - When a load-balancer runs out of resources, use a DNS round-robin mechanism to distribute traffic to different IPs [read more](https://serverfault.com/a/268939)
  - DB sharding (if too many data to be stored in a single DB)
    - Vertical (split by columns)
      - Hard, usually require application level logic
    - Horizontal (split by rows)
      - Easy, usually key mod number_of_shards
    - In addition
      - Replicate each shard
      - Add load balancer
  - In-memory Caching
    - Always read-through for sure (read cache, if miss, read DB & update cache)
    - Invalidation
      - Write-through
        - Write to cache, then write to DB
        - Pros
          - Ensure data consistency between cache & DB
          - No data loss if cache server fail
          - Last written data will be in cache
        - Cons
          - Excessive cache write even if the updated data not used later on ([cache pollution]())
      - Write-around
        - Invalidate cached item if exists, write directly to DB
        - Pros
          - Ensure data consistency between cache & DB
          - No data loss if cache server fail
          - Free up stale cache space, no excessive cache write before it is actually needed
        - Cons
          - Last updated data is not available in cache, next read will miss
      - Write-back (not recommended)
        - Write to cache, periodically copy cache to DB
        - Pros
          - Fast, no excessive write to DB
          - Last written data will be in cache
        - Cons
          - Cache and DB only eventual consistent
          - If cache fail, data loss
    - Eviction policy
      - LRU (simple, used by memcached & redis)
      - LFU (now redis also has this choice)
    
    - Decide cache location
      - One per app server (in-process cache)
        - Pros
          - Fastest, even if it is on app server's local disk (faster than through network)
        - Cons
          - If app servers are load-balanced, even if a request was previously made on a server and cached, still cache miss on different servers
          - The cached content on different servers are not guaranteed to be consistent (but will be eventual consistent when cached content timesout)
          - Each app server has limited cache capability that cannot be horizontally scaled
        - Choose if
          - The cached content are immutable (then no consistency issue)
          - Eventual consistency is acceptable
          - App is small, no need to scale, only 1 app server is sufficient
      - Shared by all app servers via network (distributed cache such as memcached, redis)
        - Pros
          - The cache is always consistent
          - Easy to horizontally scale cache
        - Cons
          - Slow due to network latency
          - A single point of failure (however, using [consistent hashing](https://www.toptal.com/big-data/consistent-hashing) algorithm, even if a cache server fail, its cached items can be distributed to other servers efficiently without rehashing all items)
        - Choose if
          - Scalability is needed
    - DB server can do some caching configuration too, this will help also
  - CDN
    - For non meta data storage, we cannot cache it in memory (too expensive)
    - Instead we make sure these data are close to the user's geographical location, content delivery network is the solution
    
- Speed
  - Your DB design may not be able to serve all requests at reasonable speed
    - Duplicate data and create a different DB structure for the slow request
    - Pre-populate the DB for slow request using a separate back-end service
  
## Security
- API
  - key (access control)
  - timestamp (replay attack)
- Load balancer (defend DDoS attack)
- Avoid incremental or easy to guess paths/keys
- Server with public IP (can be accessed from the internet) must use HTTPS. For internal IP serverse, use HTTP.

## Maintenance
- Treat maintenance as separate app/service that operates on the DB

## Logging
- Consider is your design easy to get usage metrics

## References
- [Introduction to architecting systems for scale](https://lethain.com/introduction-to-architecting-systems-for-scale/)


