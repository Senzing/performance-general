# General Performance FAQ

## How does Senzing scale?
In your application/server you initialize one G2Engine per process.  Then you share that G2Engine across all the threads in the system.  Senzing automatically creates a "context", including DB connections, for each thread you call G2Engine functions.  The G2Engine doesn't have any internal thread pool of its own.

What does this mean:
1. You need real OS threads.  Python gevents, golang Goroutine, Java Fibers, etc won't scale.  Those all just cooperatively share threads (sometimes one thread).  You can't parallelize any further than the number of OS threads you use.
2. Since each thread creates a context, you don't want to be constantly creating/destroying threads.  Typically this means a thread pool of some sort for multi-threaded programming.  There are good examples in Senzing's Quickstart for Docker for both Java and Python.  A terrible use would be Python Flask's multi-threading which fires up (and kills) a new thread for each request.

G2Engine is also a "share nothing but the database" architecture so it will generally scale until your database(s) or network can't handle additional workload.  The exception to this is when you have data contention (multiple records trying to resolve the same entity at the same time).


## High-level stats
The first thing to do when developing your own application/service is to periodically (every 5min?) output the JSON document returned from G2Engine.stats(...).  This function returns internals with no clear text data on the amount of work Senzing is doing for the process.  It is an accumulation of all the work for all the threads in the process and resets each time you call it.  8 of 10 times you have a performance issue, if you send this to Senzing support, they can tell you what is going on.

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


## Digging deep
Senzing is made for developers and because of that we have left function symbols and line numbers in the library.  This means if you get a stack trace from a running Senzing application you can know exactly what lines of code it is executing, yet without revealing any data.

### GDB
For example, with sudo access and gdb installed you can use it to dump such a stack for the entire process, even if that process is running inside a Docker container on the system.
```
--- replace 1234567 with the actual process id (PID) of the running program
$ sudo gdb -p 1234567 -batch -ex "thread apply all bt" > dump.out
```

If you look in dump.out you will see things like:
```
Thread 129 (LWP 1176259 "python3"):
#0  0x00007f2cbbf4e314 in recv () from target:/lib/x86_64-linux-gnu/libpthread.so.0
...
#13 0x00007f2cadaeb131 in ?? () from target:/opt/microsoft/msodbcsql17/lib64/libmsodbcsql-17.10.so.4.1
#14 0x00007f2cadec5373 in SQLExecute () from target:/usr/lib/x86_64-linux-gnu/libodbc.so.2
#15 0x00007f2cadf2b469 in MSSQLOperations::executeGenericStatementWithoutResult () at /mnt/workspace/buildmgr/autobuild/workspace/g2/g2_team_sprint73_202308110734/amzn2_x64_gcc103/G2/dev/libs/db/drivers/MSSQLOperations.cpp:556
#16 0x00007f2cbaabb299 in SimpleDatabaseStatementImp::execute () at /mnt/workspace/buildmgr/autobuild/workspace/g2/g2_team_sprint73_202308110734/amzn2_x64_gcc103/G2/dev/libs/db/SimpleDatabaseStatement.cpp:128
#17 0x00007f2cbaac0118 in SimpleDatabaseStatementUtil::internalExecute () at /mnt/workspace/buildmgr/autobuild/workspace/g2/g2_team_sprint73_202308110734/amzn2_x64_gcc103/G2/dev/libs/db/SimpleDatabaseStatementUtil.cpp:85
#18 0x00007f2cbaac0146 in SimpleDatabaseStatementUtil::execute () at /mnt/workspace/buildmgr/autobuild/workspace/g2/g2_team_sprint73_202308110734/amzn2_x64_gcc103/G2/dev/libs/db/SimpleDatabaseStatementUtil.cpp:41
#19 0x00007f2cba9708d0 in SQLBackingStore::InsertLIB_FEAT () at /mnt/workspace/buildmgr/autobuild/workspace/g2/g2_team_sprint73_202308110734/amzn2_x64_gcc103/G2/dev/libs/DataLayer/SQLDatabaseBackingStore.cpp:3091
#20 0x00007f2cba9081a6 in DataAccessImplementation::InsertLibFeat () at /mnt/workspace/buildmgr/autobuild/workspace/g2/g2_team_sprint73_202308110734/amzn2_x64_gcc103/G2/dev/libs/DataLayer/DataAccessImplementation_dataContent.cpp:977
#21 0x00007f2cba94a9f6 in LibFeatCache::internal__insertLibFeat () at /mnt/workspace/buildmgr/autobuild/workspace/g2/g2_team_sprint73_202308110734/amzn2_x64_gcc103/G2/dev/libs/DataLayer/LibFeatCache.cpp:326
#22 0x00007f2cba94bd45 in LibFeatCache::insertLibFeat () at /mnt/workspace/buildmgr/autobuild/workspace/g2/g2_team_sprint73_202308110734/amzn2_x64_gcc103/G2/dev/libs/DataLayer/LibFeatCache.cpp:433
#23 0x00007f2cba89d013 in LoaderManager::AddLibFeat () at /mnt/workspace/buildmgr/autobuild/workspace/g2/g2_team_sprint73_202308110734/amzn2_x64_gcc103/G2/dev/libs/loader/LoaderManager.cpp:1086
#24 0x00007f2cba89d8e8 in LoaderManager::PreprocessLibFeats () at /mnt/workspace/buildmgr/autobuild/workspace/g2/g2_team_sprint73_202308110734/amzn2_x64_gcc103/G2/dev/libs/loader/LoaderManager.cpp:1227
#25 0x00007f2cba8a0d0e in LoaderManager::AddObsEntity () at /mnt/workspace/buildmgr/autobuild/workspace/g2/g2_team_sprint73_202308110734/amzn2_x64_gcc103/G2/dev/libs/loader/LoaderManager.cpp:678
#26 0x00007f2cba8a1a79 in LoaderManager::ProcessObs () at /mnt/workspace/buildmgr/autobuild/workspace/g2/g2_team_sprint73_202308110734/amzn2_x64_gcc103/G2/dev/libs/loader/LoaderManager.cpp:517
#27 0x00007f2cba8a3304 in LoaderManager::LoadEntity () at /mnt/workspace/buildmgr/autobuild/workspace/g2/g2_team_sprint73_202308110734/amzn2_x64_gcc103/G2/dev/libs/loader/LoaderManager.cpp:422
#28 0x00007f2cba8a361b in LoaderManager::LoadObservation () at /mnt/workspace/buildmgr/autobuild/workspace/g2/g2_team_sprint73_202308110734/amzn2_x64_gcc103/G2/dev/libs/loader/LoaderManager.cpp:258
#29 0x00007f2cba6e7e89 in G2Embed::G2Processor::processWithG2 () at /mnt/workspace/buildmgr/autobuild/workspace/g2/g2_team_sprint73_202308110734/amzn2_x64_gcc103/G2/dev/apps/g2/embed.cpp:3134
#30 0x00007f2cba6ebd4b in G2Embed::G2Processor::processStatic () at /mnt/workspace/buildmgr/autobuild/workspace/g2/g2_team_sprint73_202308110734/amzn2_x64_gcc103/G2/dev/apps/g2/embed.cpp:2105
#31 G2Embed::G2Processor::process () at /mnt/workspace/buildmgr/autobuild/workspace/g2/g2_team_sprint73_202308110734/amzn2_x64_gcc103/G2/dev/apps/g2/embed.cpp:2099
#32 0x00007f2cba6ec3a9 in G2Embed::G2Processor::addOrReplaceRecord () at /mnt/workspace/buildmgr/autobuild/workspace/g2/g2_team_sprint73_202308110734/amzn2_x64_gcc103/G2/dev/apps/g2/embed.cpp:2133
#33 0x00007f2cba6ec58d in G2Embed::addOrReplaceRecord () at /mnt/workspace/buildmgr/autobuild/workspace/g2/g2_team_sprint73_202308110734/amzn2_x64_gcc103/G2/dev/apps/g2/embed.cpp:920
#34 0x00007f2cba68d80e in _internal_G2_replaceRecord () at /mnt/workspace/buildmgr/autobuild/workspace/g2/g2_team_sprint73_202308110734/amzn2_x64_gcc103/G2/dev/apps/g2/libg2.cpp:590
#35 0x00007f2cba68dcf4 in G2_addRecord () at /mnt/workspace/buildmgr/autobuild/workspace/g2/g2_team_sprint73_202308110734/amzn2_x64_gcc103/G2/dev/apps/g2/libg2.cpp:640
#36 0x00007f2cbaed6d1d in ?? () from target:/usr/lib/x86_64-linux-gnu/libffi.so.7
#37 0x00007f2cbaed6289 in ?? () from target:/usr/lib/x86_64-linux-gnu/libffi.so.7
#38 0x00007f2cbaeef360 in ?? () from target:/usr/lib/python3.9/lib-dynload/_ctypes.cpython-39-x86_64-linux-gnu.so
```

I'm not going to ask you to understand this or go though the entire file but you can see that:
1. a python program (line #38)
2. is calling the Senzing API G2_addRecord (the C name that the python SDK calls))... and look, it is on line 640 of libg2.cpp
3. that the thread is in an msodbc SQLExec call (like #14)
4. and that SQLExec was a result of the Senzing SQL datalayer (SQLBackingStore) function InsertLIB_FEAT line 3091 of SQLDatabaseBackingStore.cpp

### Making more sense of the stack
With some quick bash commands you can make short work of the analysis.

```
$ grep -P ':\d+$' dump.out | grep ' in ' | awk 'function basename(file, a, n) {
    n = split(file, a, "/")
    return a[n]
  }
{print $1" "$4" ",basename($NF)}' > summary.out
```

Running that will extract a vastly simplified stack that focuses nearly exclusively on the Senzing functions.  Looking at summary.out will show you something like this.
Note: I will also be posting a python script that does this better but is more susceptible to gdb output and symbol changes.
```
#20 DataAccessImplementation::InsertLibFeat  DataAccessImplementation_dataContent.cpp:977
#21 LibFeatCache::internal__insertLibFeat  LibFeatCache.cpp:326
#22 LibFeatCache::insertLibFeat  LibFeatCache.cpp:433
#23 LoaderManager::AddLibFeat  LoaderManager.cpp:1086
#24 LoaderManager::PreprocessLibFeats  LoaderManager.cpp:1227
#25 LoaderManager::AddObsEntity  LoaderManager.cpp:678
#26 LoaderManager::ProcessObs  LoaderManager.cpp:517
#27 LoaderManager::LoadEntity  LoaderManager.cpp:422
#28 LoaderManager::LoadObservation  LoaderManager.cpp:258
#29 G2Embed::G2Processor::processWithG2  embed.cpp:3134
#30 G2Embed::G2Processor::processStatic  embed.cpp:2105
#32 G2Embed::G2Processor::addOrReplaceRecord  embed.cpp:2133
#33 G2Embed::addOrReplaceRecord  embed.cpp:920
#34 _internal_G2_replaceRecord  libg2.cpp:590
#35 G2_addRecord  libg2.cpp:640
#15 MSSQLOperations::executeGenericStatementWithoutResult  MSSQLOperations.cpp:556
#16 SimpleDatabaseStatementImp::execute  SimpleDatabaseStatement.cpp:128
#17 SimpleDatabaseStatementUtil::internalExecute  SimpleDatabaseStatementUtil.cpp:85
#18 SimpleDatabaseStatementUtil::execute  SimpleDatabaseStatementUtil.cpp:41
#19 SQLBackingStore::InsertLIB_FEAT  SQLDatabaseBackingStore.cpp:3091
#20 DataAccessImplementation::InsertLibFeat  DataAccessImplementation_dataContent.cpp:977
#21 LibFeatCache::internal__insertLibFeat  LibFeatCache.cpp:326
#22 LibFeatCache::insertLibFeat  LibFeatCache.cpp:433
#23 LoaderManager::AddLibFeat  LoaderManager.cpp:1086
#24 LoaderManager::PreprocessLibFeats  LoaderManager.cpp:1227
#25 LoaderManager::AddObsEntity  LoaderManager.cpp:678
#26 LoaderManager::ProcessObs  LoaderManager.cpp:517
#27 LoaderManager::LoadEntity  LoaderManager.cpp:422
#28 LoaderManager::LoadObservation  LoaderManager.cpp:258
#29 G2Embed::G2Processor::processWithG2  embed.cpp:3134
#30 G2Embed::G2Processor::processStatic  embed.cpp:2105
#32 G2Embed::G2Processor::addOrReplaceRecord  embed.cpp:2133
#33 G2Embed::addOrReplaceRecord  embed.cpp:920
```

This itself isn't useful but a quick bit of awk/sort/uniq and you get something very helpful.
```
$ awk '{print $2}' summary.out | sort | uniq -c | sort -n
...
      7 ERManager::ResolveEntity
    107 DataAccessImplementation::InsertLibFeat
    107 LibFeatCache::insertLibFeat
    107 LibFeatCache::internal__insertLibFeat
    107 SQLBackingStore::InsertLIB_FEAT
    109 LoaderManager::AddLibFeat
    109 LoaderManager::PreprocessLibFeats
    111 LoaderManager::AddObsEntity
    111 MSSQLOperations::executeGenericStatementWithoutResult
    112 LoaderManager::LoadEntity
    112 LoaderManager::LoadObservation
    112 LoaderManager::ProcessObs
    116 SimpleDatabaseStatementImp::execute
    116 SimpleDatabaseStatementUtil::execute
    116 SimpleDatabaseStatementUtil::internalExecute
    124 G2Embed::G2Processor::processStatic
    124 G2Embed::G2Processor::processWithG2
    125 G2_addRecord
    125 G2Embed::addOrReplaceRecord
    125 G2Embed::G2Processor::addOrReplaceRecord
    125 _internal_G2_replaceRecord
```

This is a list of the most common functions called in the process.  Remember that each thread has many functions on its stack and there are some cases where the same function could conceivably (though rarely) be more than once on the stack.  This is why you see multiple things at the same number of instances.

A quick look and you see:
1. 125 threads are calling G2_addRecord. this aligns pretty well with the workload stats that showed 126 active threads -- they won't line up perfectly due to timing
2. 107 threads are in the SQLBackingStore::InsertLIB_FEAT.  this also aligns well with the workload stats
3. lastly, there is little else going on

Keep in mind this is the "ground truth", you can't argue with it.  gdb takes the running process and generates this stack.  This IS what the application is doing and, in this case, it is largely waiting on inserts into a particular DB table.  As we noted previously, the database(s) or network will almost always eventually become your bottleneck if you avoid data contention.

NOTE: But don't jump to conclusions too quickly on this wait! It only means that the delay could be in the database client library from the vendor, the network, or the database server itself.  It is important to take a holistic view.  In fact, if you get unixodbc from RedHat or Microsoft (not Debian/Ubuntu), the library is built with a global lock and can NOT scale via threads... such a stack like above would have nothing to do with the database or network in that case.  It also does NOT mean that the database can't scale further to handle additional load by running more threads/processes calling addRecord.


