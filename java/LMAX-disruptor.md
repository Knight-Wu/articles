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

# Use fence to avoid instruction reorder
* How to avoid instruction reorder and dirty cache line  if we not using lock ?

 The answer is to use fence. JMM ensure acquireFence can read all data that written before releaseFence.

* The order of produce and consume:
  
 ```

1. write event
2. release fence
3. write sequence(publish)

4. read sequence
5. acquire fence
6. read event
```
 
说白了就是使用了fence 的HB 语义:
如果一个线程执行了 “release 动作”，
另一个线程执行了 “acquire 动作”，
并且 acquire 读到了 release 之后的“某个共享变量” sequence，
那么 release 之前的所有写入，对 acquire 之后的线程必须可见。
