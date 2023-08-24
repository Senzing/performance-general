# General Performance FAQ

## How does Senzing scale?
In your application/server you initialize one G2Engine per process.  Then you share that G2Engine across all the threads in the system.  Senzing automatically creates a "context", including DB connections, for each thread you call G2Engine functions.  The G2Engine doesn't have any internal thread pool of its own.

What does this mean:
1. You need real OS threads.  Python gevents, golang Goroutine, Java Fibers, etc won't scale.  Those all just cooperatively share threads (sometimes one thread).  You can't parallelize any further than the number of OS threads you use.
2. Since each thread creates a context, you don't want to be constantly creating/destroying threads.  Typically this means a thread pool of some sort for multi-threaded programming.  There are good examples in Senzing's Quickstart for Docker for both Java and Python.  A terrible use would be Python Flask's multi-threading which fires up (and kills) a new thread for each request.

G2Engine is also a "share nothing but the database" architecture so it will generally scale until your database(s) or network can't handle additional workload.  The exception to this is when you have data contention (multiple records trying to resolve the same entity at the same time).


## High-level stats
The first thing to do when developing your own application/service is to periodically (every 5min?) output the JSON document returned from G2Engine.stats(...).  This function returns internals with no clear text data on the amount of work Senzing is doing for the process.  It is an accumulation of all the work for all the threads in the process and resets each time you call it.

This is mostly intended for something to be sent to support and the fields aren't generally documented and may change without notice.  Since the database(s) and network capabilities as well as data connection are the primary limiters to scaling, I will describe those parts here.
```
{ "workload": { "apiVersion": "3.8.0.23223"
...
  "retries": 27 -- this means some level of general contention, smaller the better
...
--- these next three being non-empty means there is some level of sorting/grouping of the input records, shuffling the input records is recommended
  "latchContention": [ ] -- a list of feature types and number of times contention is predicted and prevented before involving the database, empty is ideal
  "highContentionFeat": [ ] -- a list of LIB_FEAT.FEAT_HASH values that are particularly problematic, this should be empty
  "highContentionResEnt": [ ] -- specific RES_ENT_IDs that are causing contention, this should be empty
...
--- threadState is only a snapshot of what threads are doing the instant that G2Engine.stats(...) is called so you look for trends
  "threadState": {
    "active": 126, -- total number of threads in an active G2Engine call (e.g., addRecord)
    "idle": 3, -- threads not in an active call, ideally small single digits under heavy load.  If this isn't close to zero then you are having problems feeding your threads fast enough
    "sqlExecuting": 117, -- the number of threads executing SQL statements or fetching results.  This means the system is waiting on the database.
...
}
```
