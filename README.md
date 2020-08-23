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
- Availability
  - Replicate
    - Server
    - DB
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
    - Horizontal (split by rows)
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

## Security
- API
  - key (access control)
  - timestamp (replay attack)
- Load balancer (defend DDoS attack)

## Maintenance
- Treat maintenance as separate app/service that operates on the DB

## Logging
- Consider is your design easy to get usage metrics


