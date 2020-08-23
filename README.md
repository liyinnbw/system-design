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
- SQL or NoSQL?
  - Is there more relationship than a basic key-value pair?
  - Scalability?
- Schema

## Design for Functional Requirements
- Draw UML or simple workflow diagram to help understanding
  - User request, App server, DB, arrow to represent data flow
- Explore various design options, state their pros and cons before making a decision

## Design for Non-functional Requirements
- Availability & Reliability
  - Replicate
    - Server
    - DB
  - Separate Read/Write to different servers
    - Each server can handle limited connections at any single time
    - If many time-consuming writing process (such as upload) take place at the same time, read requests cannot be handled
- Scale out
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
  - DB sharding (if too many data to be stored in a single DB)
    - Vertical (split by columns)
      - Hard, usually require application level logic
    - Horizontal (split by rows)
      - Easy, usually key mod number_of_shards
    - In addition
      - Replicate each shard
      - Add load balancer
  - In-memory Caching
    - Usually read/write-through cache
    - Eviction policy
      - LRU
      - LFU (better)
    - Decide cache location
      - One per app server
        - Con: Need to update all caches when DB is updated
      - Shared by all app servers
        - Con: High stress, single point of failure
          - Solution: replicate cache server and add load balancer for them too, but need to synch all of them when DB is updated
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

## Maintenance
- Treat maintenance as separate app/service that operates on the DB

## Logging
- Consider is your design easy to get usage metrics


