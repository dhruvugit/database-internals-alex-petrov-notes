# Transaction Processing and Recovery

Till now we have learnt about the data strutures stroage engine uses now we will see, buffer management, lock, recovery which are prerequisites for transaction. 

Transaction is an indivisible logical unit of work which includes multiple operations it can be read-write both. Transaction provides some transaction guarantees, these properties are called as ACID. 

**Atomicity**  : As said transaction are indivisible which contains multiple operations, so either all have to be successfully executed and if even one of them fails, the whole transaction gets rolled back. 



**Consistency** : This is the weakest guarantee that transaction provides as it is something which is defined by the user not the database. It says transaction should bring a database from one valid state to another valid state, using database invariants like constraints, referential intergrity, etc. 


**Isolation**: Multiple transaction should be able to run at a same time as if they feel no other transaction is running and interfering with the same work. 

It defines when the database changes from a transaction may become visible and when those changes may become visible to concurrent transaction.

We also have weaker isolation levels due to performance reasons. 



**Durability**: Once a transaction is committed all database state should be persisted on disk and survive power outages, system failure, crashes. 



#### Some parties which make data persistence done: 

1. **Transaction Manager**: Coordinates and schedules the transaction and their individual steps. 

2. **Lock Manager**: It helps to avoid the data integrity by guarding access to resources. Using shared and exclusive locks. Exclusive lock can be held by just one transaction so other have to wait once the transaction is committed or lock released the waiting transaction gets notified. 

Shared Lock is Read Lock, Exclusive Lock is Write lock. 

3. **Page Cache**: It's intermediary between disk and storage engines. It holds the data or records in cache for the pages that are not yet being synchronized with the persistent storage. All the DB changes are first applied to page cache. 

4. **Log Manager**: Hold history of operations as log entries which are done to page cache. 
* They can be used to reconstruct the page cache in case of system failure. 
* Can used to undo the changes from a aborted transaction. 



## Buffer Management
Database two memory level = RAM + DISK 

Page Cached on memory when read from disk and if again same request comes gets served for the page cache instead of a new disk read. 

It assumes no other process have changed the data on disk before returning the page cache entry. This approach reffered as `Virtual Disk` or `Page Cache` or `Buffer Pool`. 


Paged In: Un-cached pages when loaded from disk. 

Dirty Page: Any new changes made to page cache which are not yet written to disk. They are no more called dirty when those changes are also written to disk. This is a bit different from which we see proper cache like Redis in software system when we say that cache is dirty because the source of truth have new data. In databases Page Cache are on more priority for recent data.


Page eviction (removal of old pages in cache) happens when the page cache is full and new page wants to enter, as actual DB storage is huge and caches are generally short on storage. 


> **Bypassing Kernel Page Cache**
> 
> Kernel: The core of OS which runs on kernel space which manages all the hardware like CPU, disk, devices. Application ask kernel in order to talk to hardware and kernel does that via systems calls. 
> 
> So when kernel reads from disk it also cache that read on local kernel cache, it's double copying in DB pov, as DB also caches the data in Page Cache. So in order to avoid copy of same data at two RAM places one can use `O_DIRECT` flag while read() system call. 


### Caching Semantics 
Changes made to buffer are kept in memory until they are eventually written back to disk. It also helps to keep the tree partially in memory and we can make changes in memory itself and then later on flush to disk. 

If the requested page is not yet cached it gets page with logical address or page number and translates to physical address on main memory. We also have option of pinning which avoid the page eviction from cache. When there are changes to page it gets marked dirty. 




### Cache Eviction 
Cache size is limited and pages need to be evicted if storage if full. If page is in sync with the disk i.e it's not dirty page and page is also not pinned in that case page can be removed from page cache. This also is done as a background batch process via `background flush writer`. 

Another important aspect is durability. If the system crashes, all the dirty pages or unflushed data will be lost. For this we use the WAL (Write Ahead Logs) to recover the cache. Only those logs can be discarded from WAL whose page cache got flushed to disk. 

Some Tradeoffs: 
1. Postpone Flush: We can postpone the flush to avoid the disk access
2. Early Flush: We can do early flush for quick eviction of them
3. Which Page to Evict: Optimal order of pages for eviction. 
4. Keep cache size within memory limits 
5. Avoid loosing data which is not yet persisted. 



### Locking Pages in Cache 
We can't do disk IO on each read write. Also in subsequent read writes chances are high that next changes go to the same page. Now as BTree gets narrower on top (from root end and below it) and split merges propagates to these so this least part of tree can be permanently cached. 


We can lock pages or pin pages which have high probability of getting used, this is called Pinning of Page or Locking of Page. And other broader nodes of the tree can be cached on demand. 

**_Note_**: Now this means for the query we don't need to make h (height of BTree) times disk access. As we have cached top level nodes, we just need some few disk reads to get the exact node. 

The database waits for a moment to collect multiple changes. Perhaps three deletes and two inserts happened on the same page. Instead of performing five separate disk writes, it calculates the final state of the page in memory and performs only one write to disk.
The system tried to do merge, split and other tree structure changes in memory itself in order to avoid the multiple disk reads. 

> **Prefetching and Immediate Eviction**
> 
>Prefetching: During range scan of a particular range, the next set of records can be prefetched. 
> 
>Immediate Eviction: Simply put, immediate eviction after the operation with that page is done, nothing fancy. 



### Page Replacement 
* Eviction is like throwing something away from a full closet.
* Page replacement is the rule you follow to decide what to throw away.



#### FIFO and LRU
* FIFO: Here we have a queue where we put the page to the tail and as soon as queue fills up, removal is done on head. This strategy won't work in heavy workloads where the root level nodes which are frequently accessed due to their top level would be required again and again. So multiple disk read, cache with such page replacement won't help. 
* LRU: This is obvious extenstion to FIFO, where we keep on arranging the pages on the basis of access, if the page in bettween is accessed again is moved to the end of the order as if it is recently paginated. But here also there is alot of reference change work which might hurt in high concurrent env. 

There is also a variation of LRU which uses 2 LRU queue one as hot another as cold. 


#### CLOCK 
1. Referenced Page: When page is in use and doing some DB operation, when it's not doing one, it's called unreferenced. 
2. Access Bit: A flag assigned by DBMS to the page when a read, write or some other operation is done on page, it's get bit `1` when there is some operation done. 

3 cases on this clock
* Referenced : Clock can not touch it for eviction it will simply ignore it
* Unreferenced,  Access Bit 1: In this case clock will see this page as recently used and will give another chance, while leaving this page it will mark it's access bit to 0. If it again comes to this page and access bit is still 0, it means for whole clock rotation it was not used hence candidate for eviction. If it's dirty page everything will be flushed and then evicted. 
* Referenced,  Access Bit 0: It's candidate for eviction. 

Note that if page is in use, clock will ignore it. Intialy the page bit will always be 1 as it would have been used by some operation, then clock sweep changes it to 0. 

#### LFU
* LRU: Focuses on recency of the data, irrespective of how many times it gets used. 
* LFU: Foucses on frequency of the data used, not recency. 
>**Important**
> 
>LFU shifts the focus from recency of access to the frequency of access. It addresses a key weakness of LRU, where a single, large table scan (a "one-hit wonder") can flush out frequently used but less-recently accessed pages (like critical index pages) from the buffer.

TinyLFU is one of the strategy used for LFU written in Java using Caffeine library. It uses multi queue architecture
* Admission, maintaining newly added elements, implemented using LRU policy.
* Probation, holding elements most likely to get evicted.
* Protected, holding elements that are to stay in the queue for a longer time



### Recovery 
In the Database world, there are many layers of hardware, software in order to make it work sound. Hence we need to consider many failure scenarios for the same. 

* Write Ahead Log (WAL or commit log) is a append only, disk-resident (it's on HDD/SSD not on RAM) structure used for crash and transaction recovery. 

> **PostgreSQL Versus fsync()**
> 
> **Checkpointing**: The process of forcing all modified (dirty) data pages from memory to be written to the physical disk files, this process relies on kernel call `fsync()`
> 
> PostgreSQL relies on the fsync() command during checkpoints to safely write modified data from memory to disk. However, on Linux, fsync() can fail to write the data but still incorrectly report success, misleading the database. This conflict can cause PostgreSQL to wrongly assume data is safe, leading to silent data loss or corruption.

#### Log Semantics 
* WAL is a immutable log, it also have a log buffer. When this log buffer fills up the transaction manager or page cache do force flush (`force` operation) to the disk for the logs. These logs have a Log Sequence Number which is monotonically increasing number attahed to each record generated using increasing number or timestamps. 
* If we have a set logs under a transaction, that transaction won't be said commited until unless the LSN with the commit record is `forced` to disk. 
* Hence WAL cache don't mean operation is commited, transaction said to be committed if all logs are on disk. 
* Making sure logs persistence is really critical, for this databases follow log checkpoints which tells upto what record logs are flushed to disk. (Read on trimming, I think it tells where to stop not sure...). Checkpoints really help in database startup to load everything. 
* WAL is coupled with a primary storage structure (log cache + disk)
* **Sync Checkpoint**: Process which force flushes all the dirty pages to the primary storage structure (disk). 
* Impractical Task: Stop all the process and transaction and flush everything for log buffer (the log cache) to the WAL file on disk, this is highly inefficient. This "stop-the-world" can not be considered. 
* `Fuzzy Checkpoint`: Process which occurs over a period of time without stopping the processes. 

Fuzzy Checkpoint process have a `begin_checkpoint` record which gets added to WAL which contains the info about the dirty pages and what to flush, async way dirty pages are then flused to disk, anyone can still modify these pages they will be handled by future checkpoints. After this a `end_checkpoint` file is written 

> **Compensation Log Records (CLRs):** 
> 
> It's purpose is to make the undo process restartable/undo and idempotent. 
> 
> It solves a problem of "what if process crashes while rolling back transaction"
> 
> It just writes a extra record before rollback change that I am changing, now in case the crash happens, in recovery phase that CLR will be considered that rollback was already performed for this uncommited transaction. 



#### Operation vs Data log 
This section talks about undo and redo operations and ways to do it.
* Shadow Paging: When page is modified it's not overwritten to orignal page, instead it write new changes to a fresh new page and stores this page pointer to the old page, any rollback will delete this new page and use the old page pointer for results. 
* Redo Operation (**Recovering from a crash**): Applies changes to before image to get the after image, helps in roll foward after a crash. 
* Undo Operation (**Rolling back a transaction**): Applies change to after image to get the before image, used for rolling back transaction. 
* Data Log (Physical Logging): Physcially store the new changes to a fresh page and facilitate which page to use as per the final transaction commit. If the final transaction commits the old page will be overwritten with the new page. Suppose you did 20 byte change to a 8 KB old page, new page will be there which same content and that 20 byte change, and all other rows will be same. After commit the old page will be overwritten with the new one. This is verbose and can generate a lot of logs. 
* Operation Log (Logical Logging): It simply stores the log as SQL statement like `UPDATE` and for undo the reverse of this `UPDATE` statement. This also might be complex as you need to apply these operation which depend on the state of the data. 

Hybrid Approach: Logical Undo + Physical Redo. For Example: 


| Scenario | What's Used | Why It's Better |
| :--- | :--- | :--- |
| **Transaction Rollback** (while DB is running) | **Logical Undo** | Efficient and precise. It performs targeted reverse operations without locking and overwriting entire data pages. |
| **Crash Recovery** (after DB restart) | **Physical Redo** | Fast and simple. It mechanically applies the final data state without re-executing complex logic, minimizing downtime. |




#### Steal and Force Policies 
**Steal Policy**: Flushing the dirty page with uncomitted transaction. Why would someone do that ? As we are on PageCache and stroage for Pages is limited on it, so in order to not make other operations wait we simply flush this uncomiited diry page keeping in mind how we are going to rollback if required using the logs. Hence steal policies prioritize system throughput. But if the system crashes and during recovery the disk now contains the pages with ghost data so DB needs to perform the undo operations using the logs, hence it must maintain UNDO logs. 

**No Steal Policy**: Very Easy, no uncommited transaction dirty pages to disk, ensures the consistent state of data. Slow perforamance as other operations need some new pages and due to storage issue needs to wait till the transaction commits. 

**FORCE Policy**: commit transaction only when pages are written to disk

**No FORCE Policy**: commit transaction to logs and commit transaction, write data to disk in background process. 

Consider a transaction T1 that modifies pages P1 and P2.

1.  **T1 starts.**
2.  **T1 modifies P1.** P1 is now dirty in the buffer pool.
3.  **The buffer pool is full.** Another transaction, T2, needs to read page P3.
    * Under a **`STEAL`** policy, the Buffer Manager is **allowed** to write the dirty page P1 (with T1's uncommitted data) to disk to make room for P3.
4.  **T1 modifies P2.** P2 is now dirty in the buffer pool.
5.  **T1 issues a `COMMIT`.**
    * Under a **`FORCE`** policy, the Transaction Manager **must** now find pages P1 and P2 in the buffer pool and write them to disk. The `COMMIT` only succeeds after these writes are confirmed.
    * Under a **`NO-FORCE`** policy, the Transaction Manager writes a `COMMIT` record to the log file and can immediately return success. P1 and P2 can be written to disk later by a background process.

STEAL (Policy by Buffer Pool Manager) is a memory management policy. It answers: "What do we do when the memory (page cache) is full?"

FORCE (Policy by Transaction manager) is a commit policy. It answers: "What do we do when a transaction commits?"




#### ARIES (Algorithm for Recovery and Isolation Exploiting Semantics)
ARIES is the practical and efficient implementation of a recovery system that uses the high-performance STEAL and NO-FORCE policies. 

* ARIES = STEAL Policy (efficient memory management) + No-Force Policy (Extremely fast commits)

* ARIES have 3 phases, first is analysis using transaction logs and find the looser transaction (not been committed) and winner transaction. Phases are Analysis, REDO, UNDO 


* It needs the REDO phase because of the NO-FORCE policy (to re-apply committed changes that weren't on disk).

* It needs the UNDO phase because of the STEAL policy (to remove uncommitted changes that were written to disk).


ARIES uses LSN, versioning for all this. It was defined in 1992 but still relevant in transaction processing. 




## Concurrency Control 
Concurrency Control is set for techniques for handling the interactions between the transactions. 

**Optimistic Concurrency Control (OCC)**: Assumes conflicts are rare. Everyone works on the same single version of the data, and the database just checks for conflicts at the very end before committing. It's a "act first, ask for permission later" strategy.


**Multiversion Concurrency Control (MVCC)**: DB maintains multiple versions of the rows. Read Write operates on different snapshots. 
* Provides time travel for your data 
* Writes don't block reads as each write creates it's own new version instead of overwritting the data. 
* Reads don't block the writers
* Conflict might arise in case of write-write in that case the conflicting transaction will be aborted. 
* Read-Read will never be a issue, quite obvious.
* We will never have a Write-Read conflict in MVCC as they always have their own version/snapshot which never changes. So suppose when TranA changes bank balance and after that TranB reads (tranA not yet committed), so TranB will read from that old snapshot and will use that only, now TranA committed the write and now TranB will see only the old origal balance, which is actually correct as well. Becasue it gave a read request to get the balance when there was no commit from tranA so it's giving correct results as per the request time it asked for the bank account balance. 
* And Write-Write conflict is really easy to understand like it's highly likely they can give conflicts as now it's not just read, both transaction read from some version and expect same version when they write, which is not possible.
* Depending on the isolation level implemented by the database system, read opera‐
  tions may or may not be allowed to access uncommitted values. Multi‐
  version concurrency can be implemented using locking, scheduling, and conflict
  resolution techniques (such as two-phase locking), or timestamp ordering. One of the
  major use cases for MVCC for implementing snapshot isolation.

**Pessimistic (Conservative) Concurrency Control (PCC)**: Uses locks to run the transactions and let the other transactions to modify the row only when the last transaction is comitted or aborted and release lock. PCC can cause deadlocks as multiple transaction can wait for each other to release the lock. 



* **Important Note**: So in pessimistic everyone reads from consistent state due to locks but in case of optimistic there is just one version of that data, suppose TranA starts writing but not yet commited and not TranB started to read it wil simply read that data (as it's simply select statement), but but if TranB is part of a large transaction and TranA commited that write now TranB will see a different version of the single version of data hence will be aborted. 



### Serializability 
> This is `correctness criterion` is not the `Isolation level` we know. 

> Schedule: A schedule is a sequence (ordering) of all operations (read, write, commit, abort) from multiple transactions, preserving the order of operations within each transaction.
* The order of operations within transaction never changes, never! Interleaving in terms of mixing the operations of different transaction so that no all operations of single transaction runs once, we try to run them concurrently but making sure the individual order of transaction do not change. 
* Serializability is a property of a schedule, not of a transaction or a database.
  It tells us whether that specific interleaving (schedule) preserves correctness — i.e., behaves the same as some serial execution.
* If you have var x = 10, TranA do x = x* 10, and TranB do x = x + 10. Doing these two transaction are not serial if you consider both cases of doing TranA-TranB or TranB-TranA both will result in different output. So you can say a schedule serilizable when you're going to get the same results. 
* **Serial** : Let the tranB done fully then only do tranA
* **Serializability**: Let's mix operations of both transaction by interleaving but make sure individual transaction order remains the same. 




### Transaction Isolation 
Defines what part of transaction and when it should be visible to other transaction. Strictly speaking it's nothing but how transactions see each other while execution and drama they perform. 


### Read and Write Anomalies 


**Dirty Read:** Transaction A starts, write somethings, TranB reads and uses it, Transaction A rollback/abort. Transaction B read the stale data. 

**Non-Repeatable Read (Fuzzy read):** T1 reads, got valueA. T2 start write and commit that value to something else and commit. Now again T1 another read in same transaction now it reads something else as T2 changed it. Hence Non Repeatable reads, within a transaction same read operation is not giving same results. 

**Phantom Read**: You ran select * from user where id >10 got 5 results, again within that transaction you hit same read query now got 3 results. That's what phantom read is, within the range the results changes within same transaction. 

**Lost Update:** T1 and T2 read value V, T commits it to new value X. Now T2 also writes and commit to value Y. Woah! write of T1 got overwritten and got lost. Hence, lost updates of other transaction. 

**Dirty Write:** T1 did a dirty read and use that value for some write and commit that. 

**Write Skew:** When a single transaction respects the invariant/constraint but the group of transaction violates them. Two concurrent transactions each withdraw 200 from separate accounts, A1 and A2, which initially total 250. Each transaction individually checks and preserves the rule that A1 + A2 ≥ 0 based on outdated data, but when both commit, the combined result is A1 = -100 and A2 = -50, violating the invariant. This anomaly is called a **write skew**, where individually valid actions together break a global constraint.


| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Lost Update | Performance | Use Case |
|-----------------|------------|---------------------|--------------|-------------|-------------|----------|
| **READ UNCOMMITTED** | ✅ Possible | ✅ Possible | ✅ Possible | ✅ Possible | ⚡ Fastest | Reporting, analytics where accuracy isn't critical |
| **READ COMMITTED** | ❌ Prevented | ✅ Possible | ✅ Possible | ✅ Possible | ⚡⚡ Fast | Default in most DBs, general web applications |
| **REPEATABLE READ** | ❌ Prevented | ❌ Prevented | ✅ Possible | ❌ Prevented | ⚡⚡⚡ Moderate | Financial calculations, reports requiring consistency |
| **SERIALIZABLE** | ❌ Prevented | ❌ Prevented | ❌ Prevented | ❌ Prevented | ⚡⚡⚡⚡ Slowest | Critical operations, strict data integrity requirements |


| Isolation Level | Shared Locks (Read)                                              | Exclusive Locks (Write) |
|-----------------|------------------------------------------------------------------|-------------------------|
| READ UNCOMMITTED | No locks                                                         | Held until commit |
| READ COMMITTED | Released after read (that's why other transaction can cause NRR) | Held until commit |
| REPEATABLE READ | Held until commit                                                | Held until commit |
| SERIALIZABLE | Range locks held until commit                                    | Held until commit |



### OCC (In Detail)
* It purely works on "act first, ask permisssion later", it focuses on fact that conflicts are rare. 
* They are suited for read heavy systems. 

**Phase 1 (Read):** 
Transaction executes it's operations but in private workspace not to original pageCahce. It creates the read-set (what all reads needs to be done in transaction) and write set (what are all write operations). 



**Phase 2 (Validation):** This is crital section which checks serializability that the current transaction is safe to commit or not, what are read set and write sets of the transaction which got commited before it. If not conflict it moves foward else aborted and discarded changes. 


**Phase 3 (Write):**  If validation suceeds the write from private write set is written on to the disk. 

Don't use OCC in write heavy operations, you'll see a lot of conflicts. 

Do further read on Backward-Oriented Validation & Forward-Oriented Validation, they are nothing but how and when you handle conflict. 


### PCC (In Detail)
* I used to thought the PCC is pure lock based thing, and never aborts transaction but seems like it's much more than that. Due to that timestamp thing it actually heavily focuses on abort too. I used to thought OCC mostly do aborts. 
* PCC assumes conflicts are likely. 
* If it detects the` future conflicts`, it immediately blocks or aborts the one of the conflicting transaction. 
* **Timestamp Ordering (TSO):** Older Transaction (lower timestamp i.e came early), Young Transaction (large timestamp). 
* The database's goal is to ensure that the final result of all the concurrent operations is the same as if the transactions were executed serially, one by one, in the order of their timestamps.
* In order to enforce this order, DB keeps two fields for each row even each field which are `max_read_timestamp` (timestamp of youngest transaction read this data) & `max_write_timestamp` (timestamp of youngest generation write this data).

#### 1. The Read Rule: Don't Read from the Future
Suppose we have just one seat available. T1 and T2 start at TS 100, now at TS 105 T1 commits and takes that one seat. Now T2 reads at 105 and see that TS(T2) < max_write_timestamp(seats) i.e 100 < 105 hence this T2 needs to be aborted. 


#### 2. The Write Rule: Don't Invalidate a Future Read
Similarly, as above if some Tran reads first and changed it's `max_read_timestamp` and after that if someone tries to write (considering both started at the same time), it will see that it has a new younger TS now so this write transaction will be aborted and needs to retry with a more younger/latest TS. 



#### 3. The Thomas Write Rule: Ignoring Obsolete Writes
* It's goal is to avoid pointless aborts. 
* If an older transaction tries to write a value that has already been overwritten by a younger transaction, just ignore the old write instead of aborting the transaction.
* Just read above point 13 times you'll get it. 
* So simply putting T1 start at 100 and T2 start at 105. But somehow T2 got lead and write the status as "VIP". Now T1 will come and try to write status as "Archived" but will see it's changed by younger transaction. In normal case you'll see T1 getting aborted. But this rule say "Wait a minute. Alice's write is from the past (TS=100). A newer transaction (TS=105) has already set a more up-to-date value. In the correct serial order, Alice's 'Archived' status would have been overwritten by the 'VIP' status anyway. Her write is obsolete."

So in PCC we have two options for impl one if 2PL using locks and another is timestamp based. This timestamp based also works when the transaction starts, don't confuse like it's timestamp based to it's similar to OCC version check it's not. 




### Lock-Based Concurrency Control

---
It's a scheme of PCC which uses locks rather than trying to serilize the schemes. Although using locks can cause contention and scalability issues. 


It follow 2PL, which is 2 Phase Locking. First phase is growing phase where all locks are aquired and transactions are written (not yet commited.). Now in shrinkig phase commit happens and locks are released. 

The **drawback of 2PL is deadlock** there are high chance that T1 is waiting for T2 to release lock  & T2 is waiting for T1 to relelase lock. This can happen when two accounts try to send money to each other in this case DB will detect the deadlock between the transactions and abort one of the transaction and someone needs to retry it now. 


#### Dead Locks 
You already know this well. Let's see ways to handle deadlocks: 
1. Introduce timeouts and abort long running transactions. 
2. You can use 2PL where in growing phase all the locks are first aquired and then actual operation starts. Although such practices can significantly limit the concurrency. 
3. Another way is using `waits-for graph`. It establishes relationship between in-flight transactions and and try to detect cycles between them, very similar to Course Schedule problem on leetcode. 
4. Also there are two techniques of how to abort which are `Wait-die` & `Wound-wait`


#### Locks 
Helps to maintain the logical consistency in cases when parallel/concurrent transactions. In transaction processing, there are types of mechanism, one is logical consistency (by locks) & other is physical consistency (by latches). 

The Main task of locks is to schedule the transactions on overlapping data sets and manage database contents not the tree structure, whereas latches focus on tree structure. 



#### Latches 
Latches guard the physical representation of the contents in leaf and internal nodes. As we know these nodes are pages. So these pages go in multiple changes while insert, update, delete, split, merges. If we won't put a latch on page multiple operations can make the data to inconsistent state such as data is present in both source and target nodes, data might not be propagated to target properly. **Also even if you're not using Locks for concurrency Latches would still be used internally, no compromise there.** 

Types of interferences between concurrent operations: 
1. Concurrent Reads
2. Concurrent Updates
3. Reading while writing 

Latches protect the computer's memory from getting corrupted, while locks protect your business logic from seeing inconsistent states.



```text
T1: Acquire LOCK on row A (logical - will hold until commit)
T1: Acquire LATCH on page containing row A
T1: Modify bytes in memory (1000 → 900)
T1: Release LATCH ← LATCH IS GONE NOW!
... T1 does other work ...
... T1 still holds the LOCK ...

T2: Tries to acquire LOCK on row A
T2: BLOCKED by lock (must wait for T1 to commit)
    ↑
    This is NOT because of the latch!
    The latch was already released microseconds ago!
```

#### Key Point:

When T2 is blocked by the lock, **there is NO latch** on the page anymore. T2 is blocked for logical consistency reasons, not because T1 is physically writing to memory.

#### Another Scenario: Two transactions, NO logical conflict
```
T1: Acquire LOCK on row A
T1: Acquire LATCH on page P (contains rows A, B, C)
T1: Modify row A bytes
T1: Release LATCH ← latch gone!

T2: Acquire LOCK on row B (different row - NO CONFLICT!)
T2: Acquire LATCH on page P (SAME PAGE as T1 used!)
T2: Modify row B bytes
T2: Release LATCH

Both transactions proceed in parallel!
```

They both locked **different rows** (no logical conflict), but they both latched the **same page** (physical protection while writing).


## Why Your Understanding Was Wrong:

You thought: Locks prevent access while latches are active.

**Reality**:
- Latches are released **immediately** (microseconds)
- Locks are held **until commit** (milliseconds/seconds)
- 99.99% of the time a lock is blocking someone, **there is no latch** anymore

* Latches: "Don't touch this memory address while I'm writing bytes"
* Locks: "Don't look at this logical data until I'm done with my transaction"

* Locks are for some mill-sec and latches are mostly for millisecond 

Transaction interacting with 10 pages under Repeatable Read isolation.

Locks: Acquired once at the start, held until COMMIT/ROLLBACK (entire transaction duration)

Latches: Acquired and released 10+ times (once per page access, held for microseconds each time)
#### Readers-writer Lock 
This thing we already know, basically on shared and exclusive locks.

|           | Shared (S) | Exclusive (X) |
|-----------|------------|---------------|
| **Shared (S)**    | ✓ Compatible | ✗ Conflict    |
| **Exclusive (X)** | ✗ Conflict   | ✗ Conflict    |

**Key:**
- ✓ = Locks can be held simultaneously
- ✗ = One transaction must wait for the other


#### Latch Crabbing  (Release parent latches early when safe)
The General way is to take latch on all the pages that come into the way but this is highly ineefificnet. So Latch Crabbing is a efficient way to do this. Also in latching we try to minimize the latch time. 
There are many obvious cases to it 
1. For a read from root to find the target child, as soon as it gets the leaf page the latch from root and path can be released. 
2. In case of insert as soon as it find the leaf page and ensures that writing to it won't progpagate strucutral changes to root i.e it have enough space. In this case latch can be released instantly. 
3. Similar for delete. 
4. This latch aquisition as a root to target path as soon as we see the changes are not required in internal nodes we will keep on releasing the latch. 

Actually this approach is kind of optimistic as most insert and delete operation do not cause the structural changes. Hence root to leaf latching might sound expensive but this happens very infrequently.

_Latch Upgrading_: Instead of: Acquiring exclusive latches pessimistically down the entire path
Use: Acquire shared latches during traversal, upgrade to exclusive only when needed
Combined Strategy in Production Systems
Modern databases (PostgreSQL 12+, MySQL 8.0+) use all three techniques together:

1. Latch crabbing: Release parent latches early when safe
2. Latch upgrading: Start with shared, upgrade to exclusive only when needed
3. Pointer chasing: Skip root latch for most operations

Result: Near-linear scaling for concurrent B-tree operations up to dozens of CPU cores.


#### BLink Trees 
The Blink-Tree is a clever solution to a fundamental concurrency bottleneck in traditional B-Trees: parent node latch contention during node splits.

_(Won't lie, I was saturated with latches and don't want to read on further optimization will cover this, when I will read this book for 2nd time)._ 


### Need to read more on 
1. Compensation Log Records (CLRs) came under `Log Semantics`