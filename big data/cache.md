# cache hotspot problem
## add upstream cache
cache in upstream layer, such as app cache layer, using the key that is different from the cache layer key.

## add sufix of the cache key
old key: abc, new key: abc_000, then will have an rehash effect on the whole cache

## monitor the hotspot 
if one key query tps is rising quickly, then push multiple replica to the upstream server or cache server, and update the routing table.

# Redis
## redis cluster
### partitioning
use consistent hash, slot = hash(key) % 16384, and map the slot to the redis instance. 

### persistent
can persist by snapshot , or log, 

#### log
log is too large , or too slow, but no lag by  the cluster state.

#### snapshot
snapshot is need to backup periodically, may have some delay between the snapshot and the cluster state.
