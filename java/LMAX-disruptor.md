# Compared to java native concurrent class, the advantage of disruptor
## No lock
* Using CAS to gain the writing cursor when using multiple producer

## CPU cache friendly
* Every event has a padding to make it fit in cpu cache

## Better latency
* GC less, apply the whole ringbuffer first, and directly insert data to the event
* Lock free, only use CAS

## Disadvantage of ArrayBlockingQueue
* Multiple consumer and producer compete one lock which is to protect the whole array

## Disadvantage of LinkedBlockingQueue
* It uses two lock, one for writing , one for reading
* Every item need to create a node , increase gc pressure. 
