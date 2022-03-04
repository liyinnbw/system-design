# System Design Considerations

## Understand Requirements
- Functional (What the system can do)
- Non-functional
  - Availability
  - Speed
  - Security
- Additional (non-standard requirements)

## System Capacity Estimation
- Traffic (Bandwidth / QPS)
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
|Scalability|Can do sharding, but lost ACID and expensive to do cross-server query (since can no longer use join)|Natively supported to horizontally scale, especially Cassendra|
|Cache friendly|May not be|Yes, since cache is NoSQL key-value pair, so cache & DB can adopt similar structure, making it easier to cache|
|Data Structure|Structured (only text data with fixed type/size), and fixed columns|Unstructured data, Cassendra is 2D key-value (inner keys (columns) need not be same), MongoDB is basically json objects with no fixed schema (though you can impose schema)|
|Schema flexibility|Schema should be determined beforehand, hard to change later|Schema can be easily changed later (Cassendra can change secondary keys, MongoDB can change everything)|
|ACID (atomicity, consistency, isolation, durability) compliance|ACID (usecase: money transfer)|Usually only AID, with eventual consistency|
|Data analysis|Easy due to structured data & SQL language, but become very slow when table size grow|Harder, due to unstructured data, but still ok (Cassendra can use Hadoop/Spark to do aggregation, MongoDB has native aggregation support, but not scalable)|


## Design for Functional Requirements
- Draw UML or simple workflow diagram to help understanding
  - User request, App server, DB, arrow to represent data flow
- Explore various design options, state their pros and cons before making a decision

## Design for Non-functional Requirements
- Concurrency
  - DB transaction isolation levels
    - Read Uncommitted (no isolation) one transaction may read not yet committed writes made by other transaction, thereby allowing dirty reads
    - Read Committed (write lock on current row) guarantees that any data read is committed at the moment it is read. The transaction holds a read or write lock on the current row, and thus prevent other transactions from reading, writing or deleting it. But only applies to current row, resulting in non-repeatable reads
    - Repeatable Read (write lock on all rows) transaction holds read locks on all rows it references and writes locks on all rows it inserts, updates, or deletes. Since other transaction cannot read, update or delete these rows, consequently it avoids non-repeatable read. However it allows insertion of new rows
    - Serializable (highest isolation) concurrent transactions are performed as if they were sequentially performed, read/write lock all rows and no insertion

- Availability & Reliability
  - Replicate
    - Load balancer
      - Use a pair (HA peers)
      - Each keep track of the other's status periodically (heartbeat)
      - If one fails, the other takes over in seconds (either through on-premise floating IP mechanism, or a cloud TCP/UDP router [read more](https://cloud.google.com/solutions/best-practices-floating-ip-addresses))
    - Server (behind load balancer)
    - DB
      - MongoDB: master (read-write) & slave (read only)
      - Cassendra: all nodes equal, all nodes contains duplicated data from some other nodes (can read/write even if some nodes are down)
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
  - DB partitioning (distributed DB)
    - Vertical (split by columns)
        - Hard, usually require application level logic
        - When number of rows grow, have horizontally partition anayway
    - Horizontal (split by rows, aka [sharding]())
        - Easy, use [consistent hashing](#consistent-hashing) of key
        - Key bits are determined by the expected data rows
        - Keys can be generated using a separate service, when queried, the service moves a key from unused DB to usedDB
          - Pros
            - Separate responsibility, Apps & DB servers don't have to worry about key generation
            - No duplicated keys from distributed Apps & DBs
            - Fast, can precompute and store on disk
          - Cons
            - Single point of failure (solved by duplicating keygen services and DBs, at least two, each holding a different set of keys, can be simply even/odd keys)
        - Keys can be also generated using Snowflake Algo (but distribution may not be uniform)
    - In addition
      - Each partition can be duplicated and made highly available with a load-balancer (master-slave)
    - Cons of partitioning
      - SQL Join query can't be efficiently performed
      - SQL Referential integrity (foreign key constraints) is lost
      - DB may not be evenly distributed (some DB getd more requests), require rebalancing which is painful
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
      - LRU (simple, used by memcached & redis, most popular)
        - use map to find item node in O(1) time
        - use double-linked list to update item recency in O(1) time
          - move accessed item to head
          - evict last item if size limit exceeded
      - LFU (now redis also has this choice)
        - use map to find item node in O(1) time
        - use double-linked list to update item frequency order in O(1) time
          - increment accessed item frequncy & bubble to front if getting more frequent
          - evict last item if size limit exceeded

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
          - A single point of failure (however, using [consistent hashing](#consistent-hashing) algorithm, even if a cache server fail, its cached items can be distributed to other servers efficiently without rehashing all items)
        - Choose if
          - Scalability is needed
    - DB server can do some caching configuration too, this will help also
  - CDN
    - For non meta data (large static file like image/video) storage, we cannot cache it in memory (too expensive)
    - Instead we make sure these data are close to the user's geographical location, content delivery network is the solution
    - Some considerations:
      - CDN is expensive too, make sure only cache what's frequently accessed
      - time to live (TTL): how long the content in CDN will be outdated and need to be refetched from server
      - CDN can fail, if that happens, make sure client will request from server directly

- Speed
  - Your DB design may not be able to serve all requests at reasonable speed
    - Duplicate data and create a different DB structure for the slow request
    - Pre-populate the DB for slow request using a separate back-end service

## Security
- API
  - auth token (access control)
  - timestamp (replay attack)
- Load balancer (defend DDoS attack)
- Avoid incremental or easy to guess paths/keys
- Server with public IP (can be accessed from the internet) must use HTTPS. For internal IP serverse, use HTTP.
- Use rate-limiting if needed (avoid DDoS attack)

## Maintenance
- Treat maintenance as separate app/service that operates on the DB

## Monitoring
- Consider is your design easy to get usage metrics
  - Daily read/writes requests and data traffic
  - Peak traffic, peak time
  - Average service latencies


## References
- [Introduction to architecting systems for scale](https://lethain.com/introduction-to-architecting-systems-for-scale/)


## Useful Concepts
### [Consistent Hashing](https://www.toptal.com/big-data/consistent-hashing)
- Basic form:
  - hash each server Si to a position uniformly on a circle (360/N*i) and remember their positions (note the server position won't change when server is added or removed)
  - hash each object Oi to a position on the same circle
  - find the nearest server clockwise from Oi and save Oi to that server
  - when a server is removed:
    - move all its contents to the nearest next server clockwise
    - cons:
      - the next server will be overloaded
      - new object distribution will not be even (there is a gap between the removed server's neighbouring servers)
  - when a server is added:
    - insert the server to a position on the circle
    - new objects nearer to this new server will be added to the server
    - cons:
      - new object distribution will not be even
- Better form:
  - instead of hashing each server to a single position on the circle, hash each uniformly to k positions (k = 100-200 in practice) and remember these positions (k*N positions in total)
  - when a server is removed:
    - its contents will be distributed more evenly to remaining servers depending on object positions
    - new object distribution will still be more or less even given there are still many positions on the circle
  - when a server is added:
    - just add k more positions unfiromly on the circle for the newly added server
    - new object distribution will still be more or less even given there are still many positions on the circle
    - cons:
      - new server will always have less objects (unless you scan all existing objects and reassign them to new server)
- Rendezvous or highest random weight (HRW) hashing
  - better than consistent hashing
    - more uniform object distribution
    - just need 1 hash function and no need to precompute server positions
  - algorithm:
    - for an incoming object Oi, compute wi = hash(Oi, Si) for all Si, save Oi to Si with max wi value
    - when a server is removed:
      - for each Oi from the removed server, recompute wi among remaining servers and move Oi to max wi server
    - when a server is added:
      - just one more server to consider when adding new objects

### [CAP](https://en.wikipedia.org/wiki/CAP_theorem)
- consistency, availability, partition tolerance to network failure cannot be all achieved
- In realworld, partition tolerance is a must, so usually systems are either
  - AP: sacrifice strong consistency for high availability (still can do eventual consistency)
  - CP: sacrifice availability for strong consistency (because to ensure strong consistency, when write, need to wait for all nodes to sync before can read/write again)


### [CDN]()
- Useful for caching large media files geographically near user, also known as "edge server"
- Cons: expensive. ways to reduce cost include
  - Only cache the most popular content in CDN (using 80-20 rule)
  - Reduce content size: compression, sampling
- TTL (time to live) very important parameter to tune
- Failure: need to handle CDN failure in app code, when that happens, directly query DB server

### [DNS]()
- For some system like crawler, consider caching URL to IP mapping on your server cache to reduce DNS request
- If DNS mapping is cached, make sure it will expire since DNS mapping could constantly change

### [Replication]()
- Whenever replicate, you should think about consistency
  - First think if system can tolerate eventual consistency.
  - If not, means system needs strong consistency (like writes in a banking app), then you must wait for all replica return ack before proceed with read/another write.
- Replication is usually used in cross-datacenter DB/Cache
- Replication can:
  - reduce read latency (parallel read, regional read), at the cost of higher write latency (if strong consistency needed, if eventual consistency not much impact)
  - failure recovery (promote a slave to a master)

### [Persistence]()
- Other than replication, persisting memory data is another way to recover from failure
- Only applicable for cache data (DB data just need replication to be protected from disk failure)
- Persistence can be either to a DB or just operational logs (or document DB)

### [Scaling]()
- DB Scaling:
  - Sharding
    - celebrity problem (hot keys to be placed on different DB servers)
    - consistent hashing (to determine which key on which DB server, efficient for adding/removing servers)
  - Denormalization
    - Note once you start sharding, join won't work and manual joining in code will be expensive
    - So either you split the related fields to the same server (which usually can't be achieved due to reference from other servers) or you denormalize the required data to the same table (for example denormlize the user info to the label table in data label system)

### [HTTP response codes]()
- 301: redirect (permant, so that browser will cache the redirect url and directly request that afterwards)
- 302: redirect (temporary, subsequent call will still go to original URL)
- 403: no persmisison (needed by system with authentication)
- 429: too many requests (needed by system with rate-limiting)


### [Rate Limiting]()
- usually implemented in gateway, can be implemented in backend API server too as a middleware
- can be limited by:
  - user (application layer, rate limiting & authentication must be on the same server)
  - ip (network layer)
- algorithms:
  - token bucket (params: bucket size, refill rate)
  - leaky token bucket (params: queue size, consumer rate)
- cache
  - rate limiting states (counters, or queue), and rules should be kept in memeory
  - for distributed servers, needs a centralized cache
- HTTP:
  - rate limit response code: 429
  - rate limit HTTP headers:
    - X-Ratelimit-Remaining
    - X-Ratelimit-Limit
    - X-Ratelimit-Retry-After

### [Data compare]()
- hashing (checksum) and compare the hash
- Since hashing big data takes long, we can:
  - split big data to smaller chuncks, and build hash tree (basically hash all the way up till reaching root)

### [Distributed Unique IDs]()
- UUID (if ok with 128 bits, and no need to be strictly increasing with time)
- snowflake (if need increasing with time)
  - timestamp+datacenter+machine+sequence
  - timestamp is an offset from certain time (41 bits for 69 years)
  - can adjust other parts according to needs
  - time across servers must be network synced

### [Bloom Filter]()
- low-memory & fast way to check if item is seen with some confidence
  - basically, use N hashes & N arrays, each array has limited length
  - given an item, try hash it with N hashes, see if the corresponding element in all arrays are 1 or 0:
    - if all 1, the item is LIKELY to be seen before
    - else, the item is DEFINITELY not seen, turn the hashed elements to 1

### [Notification Service]()
- producer->queue->consumer->3rd-party-notificaton-service->client
- consumer can store the result in cache (for fast retrieval) & persist to DB (in case client is offline)
- consumer may need to log all messages for re-sending in case client doesn't acknowledge
- consumer can notify client device in two ways:
  - client device do long polling
  - consumer server & client device connect via websocket, so that server can send message to user too
- we cannot ensure message is delivered only once

### [Keep-alive]()
- include this in HTTP header, so that after initial handshakes, connection will stay between client & server
- Usually used in chat service

### [Stateful vs Stateless servers]()
- usually use stateless server, it makes design simpler
- if need stateful server, need to remember state in:
  - load-balancer (always route to the same server)
    - use zookeeper
  - cache server (always route to the same cache)
  - DB server (always query the same DB)


### [Cassendra]()
- Two dimensional key-value store (aka wide-columnar store)
- NoSQL
- All nodes equal (no single master)
- Can read/write when some nodes fail (not in strong consistency mode)
- Eventual consistency (but can be strong consistency also)
- Fast for primary indexing (outer dimension key)
- Does not support secondary index, if needed, MongoDB might be better
- Can use CQL (an SQL-like) query language
- Aggregation need extra framework like Hadoop/Spark
- Conclusion: first pick for scale!


### [MongoDB]()
- Unstructured data store (aka Document store, actually each record is a binary json object)
- Support object-oriented query language (like ORM)
- Single master node
- write must go through master and propagate to slaves (eventual consistency)
- slaves can only be read
- Not as scalable as Cassendra, need sharding (needs some setup)
- Aggregation is natively supported (but not efficient for big data)
- Conclusion: usually don't use for big scale, even MySQL is better


### [Heartbeat]()
- used to check if server is down


### [Zookeeper]()
- for stateful connection between server/client
- for bring up new server if one failed allocate to user

### [Trie]()
- primarily used for search suggestion
- Tree structure can be stored in:
  - MongoDB (due to its support for deep nesting structure & secondary index)
  - Cassendar (break the tree into nodes, better for scalability)

### [File Split]()
- split large file into chunks
- good for:
  - edit & save (just save the changed chunk, use checksum to check if chunk changed)
  - parallel upload


### [Long tail problem]()
- most user are outliers (80%)
  - do not read data often
  - do not write data often


### [Cold Storage]()
- Infrequently accessed data can even be archived
