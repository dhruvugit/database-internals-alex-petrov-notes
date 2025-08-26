* Part 1 includes chapter 1-7, mention the names if required. 

### Storage Engines 

* Database systems consist of multiple modular components: a transport layer to accept request, query processor to understand the query user gives and best way to run it, an execution engine do other operations and storage engine does the storage part of it 
* The storage engine (or database engine) is a software component of a database management system responsible for storing, retrieving, and managing data in memory
  and on disk
* Database systems provided pluggable storage engines as per different technical requirements. For example: MySQL have InnoDB (Good for ACID), MyISAM (good for heavy reads poor for transactions). Mongo DB have WiredTiger and so on.

### Variables to compare the database: 
1. Schema and record sizes
2. Number of clients
3. Types of queries and access patterns
4. Rates of the read and write queries
5. Expected changes in any of these variables

Above variables can help us to answer these questions: 

* Does the database support the required queries?
* Is this database able to handle the amount of data we’re planning to store?
* How many read and write operations can a single node handle?
* How many nodes should the system have?
* How do we expand the cluster given the expected growth rate?
* What is the maintenance process?

---

TPC-C is an On-Line Transaction Processing Benchmark which have a set of benchmarks which helps to decide the database vendor as per requirement and simmulate common application workloads. 

Benchmark is based upon the performance and correctness of the database also following the concurrent transactions. One of the main performance indicator is **_throughput_**, it tells number of request a database can process in a minute 


### Things we do in general while implementing storage engines. 
1. Design Physical data layout and organize pointers. 
2. serialization format 
3. decide how data is going to be garbage collected. 
4. make sure it works in concurrent environment 
5. make sure you never lose your data once commited. 


### Conclusion : Storage Engines. 

