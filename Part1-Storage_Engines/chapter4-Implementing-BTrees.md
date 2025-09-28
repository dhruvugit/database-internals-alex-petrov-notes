# Implementing B-Trees

After studying about file formats pages, cells, slotted pages which mainly inlcude the strcture fomat for our BTrees now we will see how to implemement them. 

It includes:
1. Organization:

            a. relationship between keys and pointers 
            b. implement headers and link between pages 

2. s processes that occur during root-to-leaf descends, namely how to
   perform binary search and how to collect breadcrumbs

Breadcrumbs = a small stack (or trail) that records the path from root to current node in a B-tree. It’s vital for efficient inserts, deletes, and range scans in storage engines.

3. Optimization techniques (rebalancing, right-only appends, and bulk
   loading), maintenance processes, and garbage collection.




## Page Header

---
Page Headers holds the information and other metadeta like layout, content, number of cells, upper lower offset ranges 
* Postgres: stores page size and layout version in header. 
* SQL Lite: Number of cells and right most pointer. 

### Magic Numbers 

It's a multibyte block, containing a constant value which signals kind of page, block it belongs to, identify version. 
* It's kind of identifier to block or page. 
*  the magic number acts as a constant, reliable seal of authenticity on every single page, checked every time the page is read from the disk to ensure the database isn't working with corrupted or incorrect data.
* They are specific to page type, all index pages will have same magic number, all data pages will have magic number and so on .

This is how it looks on page, first value is magic number for sanity check and let the reader know it's reading from correct location. 
```text
[ 0xDEADBEEF | Other Header Info | 'John Doe, 456 Oak Ave' ... ]
```


### Sibling Links
These are pointers which gets stored in page header they store the pointer to it's left and right sibling on the same level of the tree. 
* This enables fast range queries and sequential scans 
* The node don't need to go to the parent then to it's right child for adjacent pages. 
* Disadvantages is that while page split of merge database must perform extra work in order to make correct chain of doubly linked list (Read more on this at "Blink Trees" Chapter 6)


### Rightmost Pointers & Node High Keys

Suppose we have two keys in a node, 100 and 500. As node have 2 keys it must have 3 pointer for it's child.
* Pointer 1, less than 100 
* Pointer 2, between 100 and less than 500 
* Pointer 3, 500 or greater. 

So now you can't just store `(key, pointer)` as we have more pointer than keys. 

##### Method 1: The Separate "Rightmost Pointer"
This is approach for Rightmost pointer where the first N keys and N pointers are paired up and the last one is put as separator pointer in the page header. 
* The node stores its main data: `(key=100, pointer_1)` and `(key=500, pointer_2)`.
* And the 3rd pointer will be stored as is in page header without key. 
* Now when read with id 800 will come, it will see the id is greater than all avaialble keys so it will go to the rightmost pointer in the page header. 
* Hence it creates a special case when one right most poionter lives differently in header. 

#### Method 2: The "High Key" (B-link Tree)
This particular approach is adopted by Postgres they feel it's messy to store the special pointer. So they created a dummy key which do not exist in reality. 
* Suppose you know you db won't have a key greater than or equal to 1000. 
* So you create a dummy key as upper bound. 
* Now you have N+1 pointer and N+1 keys both. 
* Now nothing needs to be stored as a special key in header. 

Note: You can read why` High key` is preferred in high concurrent systems. 


### Overflow Pages
 * Node sizes are fixed and do not change dynamically. 
 * BTree algo specify only fixed number of records per page. 
 * If we have tiny space left and we have more to insert for that record in that case we might need to do page split which will cost us copying already written data to new region which is impractical for each insert of such type. 
 * To solve this issue we can build a nodes from multiple linked pages. Suppose default size is 4KB we can make multiple such pages and link to the data. These linked extension pages are called **overflow pages**. 
 * Overflow pages equally need a bookkeeping just like primary pages, they can also face fragmentation. 
 * Page ID gets added to primary page when a new overflow page is formed if another overflow page is required it's page ID is added to the first overflow page and so on. 



## Binary Search

---
This we have already discussed in previous chapter. 
* Once we get the correct node, we move forward with searching `search key` using binary search, that's why those offsets needs to be sorted in slotted array. If not you need to do linear search. 
* If it returns a negative value, it means the value does not exist, and return a insertion point. Inserting data to this insertion point will maintain the sort order. 
* If positive, we get the correct data record.

### Binary Search with Indirection Pointers 
This simply explained the binary search algo by starting with mid offset and moving left or right as per search key. 
* The B-Tree stores the main keys of your records. It guides you to the page where your record resides, and once you reach that page, you perform a binary search on the slotted array to find the offset of the required data record.



## Propagating Splits and Merges

--- 
* Splits and merges can propagate to parent nodes, so for that we need to have parent pointer too in our nodes. 
* Parent pointers needs to be updated too while split and merge or rebalancing. 
* WiredTiger of MongoDB uses parent pointer for sibling traversal in order to avoid deadlocks (Need to read the reason behind deadlock with doubly linked list way). It goes to parent and then parent to it's right child to get the sibling using recursive logic.

### Breadcrumbs
This states the fundamental problem of BTree which is `Navigating a B-Tree Backwards`
* In general, when you need to insert and no space is available on page, you need to do a split , and when you do delete operation you do merge to collect the space. In these operation, we need to propagete to parent node. 
* There are two ways to solve this problem: 

#### 1. Permanent Parent Pointers
* This approach may sound fancy and spot on where we keep on track the pointers to parent, but now we need to allocate extra space to our page, we need to make sure each page gets parent pointer updated in case of merge or split. This give very high write overhead on database. 


#### 2. Temporary Path Tracking (Breadcrumbs)
* This is a stateless approach, we are not storing anything on the BTree nodes instead during our first traversal from root to leaf we keep on storing the node pointer from root to leaf in a stack till leaf's parent. 
* The general data structure used for this is stack, Postgres call it Bstack. The top of the stack stores the immediate parent to leaf node and so on other grandparents. 


**Important Point:** 
1.  Breadcrumb stack is part of temporary, in-memory execution plan, there is nothing on disk for breadcrumbs stack. 
2. Once the INSERT or DELETE operation is complete—whether it involved a split or not—the execution plan and the entire breadcrumb stack are simply discarded from memory. The only thing that remains is the final change to the data pages on the disk.





## Rebalancing

--- 

Rebalancing tries to postpone the split and merge until it's not the only option we have. 
* It moves the elements from more occupied node to a less occupied one, this helps to avoid merge, split & also leads to less tree levels resulting in faster search but at a cost of rebalancing operation. 
* Also when both nodes gets filled, instead of splitting each to two nodes, it splits two nodes into 3 final nodes and split data into 2/3 each. Now again does that rebalancing as per requirement. 
* SQL Lite implements balance-siblings algorithm which is close to what we discussed. 



## Right Only Appends

---

This is optimization is commonly used for high write workloads when we are inserting rows with monotonically auto incrementing keys. 

* In traditional approach database incoming key can come in any order and in order to maintain the sort order, we need to traverse through BTree to get it inserted in between. 
* This Right Only Append is a special case when we know incoming key is largest till now and can safely be added to right most page.

#### PostgreSQL's "fastpath"  (The Search-Skip Optimization): 
DB Engine keeps path to the right most pointer and checks if incoming key is greater than the one on right most page and cache page have enough space to entertain it, if answer is yes, engine skips the entire tree traversal and adds record to the cache page and flush to disk later. If answer if no, it do the tree traversal and do traditional insertion. 


#### SQLite's "quickbalance" (The Split Optimization)
When the right most page is full, instead of performing a split it adds a new fresh page as rightmost page and add data to it, it assumes that as keys are monotonically increasing, page will eventually fill up. 

### Bulk Loading 
 * The Traditional approach is that when you have data in hand which is sorted and you need to create index on it. You may need to add the first row and assign to btree node and so on to all the rows in tables. 
 * Now better approach is that as data is presorted, we will try to create the BTree from bottom to up, whereas in traditional we build from root. 
 * Now we will start writing data to pages as it's sorted just keep filling the pages and keep creating the new one. 
 * Now we have all the leaf nodes and they will have their upper bound and lower bound in page header with the help of that keep on creating the parent nodes of them which are index nodes, leaf ones are data nodes. 
 * This is a very fast way of creating index using bulk load this takes help of right only append as data was pre-sorted. 


## Compression 

---
* Compression is most classical way to save space while storing things on disk. 
* But trade-off is between access speed and compression ratio. 
* Higher the compression ratio can improve size (fit data into smaller space) but comes with a cost of RAM and CPU cycles to compress and decompress it. 


#### Levels of Compression (Granularity)
Explains how big chunk we can and should compress. 

1. File-Level Compression (The Impractical Approach): It's like zipping entire database and unzip for each operation, totally impractical. 
2. Page-Level Compression (A Common & Practical Approach): Reads compressed page load to RAM, decompress the page make changes to page, decompress it, write again to disk.

Page level compression comes up with a disadvantage: As we were fetching data on the fixed page size, now compressed page of size 8KB can become of 3KB and suppose we need to fetch in page size of 4KB, now we need to fetch the 1KB from next page which is wasteful. 

3. Data-Only Compression (A Decoupled Approach): 

* Row level compression: We will compress the record in our fix size page and we can add more records to our page. 
* Column level: This is mostly used with column Dbs, where column with low cardinality can compress very well. 

#### Compression Library
There are many like Snappy, zLib, or LZ4
* For Speed (Snappy, LZ4): Hurts CPU less, fast compress decompress, not the best compression ratio 
* For Size (zLib, Zstandard): Hurts CPU, best compression ratios, best for very expensive disk space. 




## Vacuum and Maintenance

---

Talks about background task that happen in parallel with queries that maintain storage
integrity, reclaim space, reduce overhead, and keep pages in order

During update and delete, they can leave the logical space but not the contiguous space, this is what fragmentation is. We might not be able to add records between those small spaces. 
Hence we get the non-addressable data (tombstoned) and needs to be compacted. 

### Fragmentation Caused by Updates and Deletes
 * Small space which gets created after update or delete where we can't even fit the new data.And also if page have enough space at tend we can't even do split merge so that space is getting waste. 
* Upade creates more dead cells, as it writes updated data at new location and tombstone the old one which creates more fragmentation. 
* Main issue is not space but space in small fragmented size which is of no use. 



### Page Defragmentation
It's asynchronous background process process which reclaims the fragmented space, process have different names as compac‐
tion, vacuum, or just maintenance. 

* Garbage Collection: Scans pages and remove the dead cells permanently.
* Compaction: Arranges the remaining cell in correct logical order. 





---

---

---
### Need to read more on the following...
1. 