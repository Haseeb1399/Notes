
#### 12th Jan 

Why are we leaking table info

Global Alpha will remain pretty large 


Look at existing plaintext database

Existing SQL --> K/V Store 

We are designing a wrapper basically, SQL for K/V Store. 

Primarily: Why are we choosing one approach over another when it comes to queries. 

Understand what is leaked in a plaintext database within the wrapper. 
Then we can go and change up the wrapper. 


Understand how selections, range and joins are done in K/V plaintext databases. 

Google Spanner, CockroachDB

Goal: Extract these ideas and then see how to adjust waffle. 


<hr>

#### 19th Jan 

* How range queries on non-unique columns are done. 
	* Shouldn't be a linear scan. 

Ans:  It is a full-table scan in case of no indexes.
https://www.cockroachlabs.com/docs/stable/sql-tuning-with-explain
https://www.cockroachlabs.com/blog/index-selection-cockroachdb-2/
![[Pasted image 20240125155037.png]]

if you don't have index on name, query takes 981ms, if you do then it takes 4ms. 
![[Pasted image 20240125155357.png]]


* How linear scan is done. 
	* Use primary key index on the table. 

* ~~Google Scholar --> Check citations
* ~~What is cited in these Papers (ObliDB and SEAL) --> Check 

* Compare the way they are doing their range search and joins on non-unique values with what we have proposed. 
Joins: https://www.cockroachlabs.com/docs/stable/joins#inner-joins
	* Merge Join: 
		![[Pasted image 20240125220633.png]]
	* Hash Join: 
		* No index is present, CRDB will hash the smaller table into memory, scan the larger table, looking up each row in the table. 

Range: We have already discussed, If there is an index, then the index is sorted and we can easily find which doc_ids satisfy range, otherwise do a full table lookup. 


__________
How we are doing it: 
* Range: Look at index tree on proxy, issue request to index for doc_ids, fetch doc_ids. 
	* We need to incur extra round for security (Decrypting Doc_Ids) and appending fake request. 
* Joins: If pre-compute index is there, issue a request to proxy for doc_ids, fetch doc_ids. If no index then stream both tables to proxy and do join locally. 

_______ 

* Think about One large monolithic K/V Store or Separate K/V store. 
	If you make separate K/V Stores: 
		The adversary can now make graphs like this:
		![[Pasted image 20240126101843.png]]
		So it can keep a record of Alphas per table. The adversary would still know the best and worst alpha values across tables. 

	If you make a single K/V Store: 
	* Alpha will grow large as the N increases. This is under the assumption that we keep one batch for all k/v pairs for all tables.
	* A potential fix for this can be that we keep the same K/V Store (One large Space), but issue distinct batches for each table to the same K/V Space. 
	* For example, Table-A has 200,000 rows and Table-B has 100,000 rows. We could just add everything to the table, and say given B=5000, R=1000, Fd = 2000, D=50,000 
		Alpha = 150. 

		Now instead of doing just one batch, we do 2 batches, one for each table, the alpha is: 
			Table-A = 100
			Table-B = 50 
		 The point of doing this is so that the histogram isn't as tail heavy, and more flatter? (Does this really give us a benefit?)
		  Might help with performance? (Instead of pushing everything to a single 5k batch, have a 5k batch for each table). 
		  Question: What if the requested item has only one doc_id? How are we waiting for more requests? (In an SQL setting). 
		Can we guarantee that since this each individual waffle is oblivious, the whole system will be oblivious, under uniform and skewed datasets? 


Joins can be more expensive in this:
	One tables batch is full, with others is still not. 
	Keep track of which keys are from which table in order to assign batches. 

Index itself also can grow larger and larger, where we will need alpha to be quite large. Question, does the index access pattern leak anything? 

<hr> 

#### 26 Jan

* Assume R is always there. 
* Shouldn't leak complete table info;
	* Leak relative sizes
	* SQL Schema. 
* Performance gain by partition.
	* Case 1: Run waffle as is. 
	* Case 2: Run waffle with partitions data (3 Waffles, 3 Tables). 

There are M tables, 

Everything is treated as a single Monotolithic table 

Other extreme, we have M partitions of the data. 
Each Partition is each waffle partition. 
Each partition has same size, can contain multiple tables. 
Partition = x and M = # of tables, 


How many partition you create, you need to balance the size. 

1) Think about experiment, (Simple as you can)
2) Think about partitioning the tables from one monolithic map (N=0) to N= some M where there are M partitions. The size of each partition needs to be the same. 

<hr>

## Feb 2nd 

1) Index sizes --> Real world Workloads and their associated Indexes
	1) TPC-C 
	2) Chicago Crime Dataset 
	3) Health Care Dataset
	4) Deploy to CRDB, make index and then check size (on diff index values)
2) Meta Proxy --> Data Proxy --> Partitions
3) Monolithic Index, What can we do about it? 
	1) Make a new partition for index as well, Think about that too.


Feedback on Presentation: 

Segmenting between topics. 
Complete something | Pause for people to digest| 
Think about something, Take a pause and then think about what your going to say. 


<hr> 

Z = # of DISTINCT values / total rows  * 100 
#### Index Sizes: 
###### TPC-C (Scale Factor = 50) -- 5GB db size

Customer Table: 886MB (1.5M rows)
	1) Index on first_name = 45MB (B Tree) --> z = 99%
	2) Index on state = 10MB --> z = 0.045% (Around 676 unique values)
	3) Index on c_id = 11MB --> z = 0.2 (3000 unique values)

I think difference exists because the first_name is 16 bytes, where as c_id would be 4 bytes and state would be 2 bytes. 

Items: 10MB (100k rows):
	1) Index on primary key = 2.2 MB

Order_Line: 1.47GB (15M rows)
	1) Index on ol_number: 15 Unique Values: 99MB  z<<0.1
	2) Index on d_index: 10 Unique Values: 99MB Z<< 0.1
	3) Index on ol_supply_w_id: 50 Unique Values: 99MB Z<< 0.1

Stock: 1.6GB (5M rows):
	1) Index on s_w_id: 50 Unique Values: 33MB z=0.001%
	2) Index on s_i_id: 100k Unique Values: 34MB; z=2%

I think the B tree sizes for each table stays relatively the same? 

![[Screenshot from 2024-02-07 16-22-43.png]]

#### Seats dataset (5.5GB)

1) Flights Table (1.9M rows): 816MB
	1) Index on flight_id (unique): 200MB -- Check type of id. 
	2) Index on status (1 value): 13MB 
	3) Index on seats_left (11 values): 13 MB

2) Airport Table (2.6M Rows): 1.06 GB
	1) Index on c_id (unique for each row): 123MB 
	2) index on c_base_ap_id (286 unique values):  17MB 


Difference can also exist in what the primary key is (Single or a Composite key), in these benchmarks, some tables have composite keys. 

Difference between flight and tpc-c primary key is that the seats f_id is a varchar of max 77 bytes while c_firstname is max 16 bytes

## Feb 12  

##### Questions:
Reimplement SQL to KV mapping for this design. 
Is a separated Index going to give a big advantage or not.


#### Types of Queries: 
1) Point
2) Joins.
3) Range Queries 
4) Aggregate Queries (SUM, MAX,MIN, COUNT, DISTINCT?)


**![](https://lh7-us.googleusercontent.com/HdYbQ_lZ4LNGkN47WNzKEyL3pYRD33EV3tQ1EKesshKgmpZng03jg8aJ2AmNPJcjoa__RrWdCoeEyeD5u1lCyEkCxfgrg3r2bcEFTZkePUVEN6z3tbZ298gSQmEQwOSe2wKbtC7HLaem9wrJwFCkSog)

#### Breakdown (Only Real Key Logic)

#### Point Queries:

1) Point Queries: 
	1) Initialization: `CreateTable(Name=tableName,Data=data,Index=[Col1,Col2])`:
		* Index proxy will call SET(Col1) and SET(Col2) on the data provided to the by the user. Using the values from SET calls, it will initialize two BSTS (real keys) for both of the indexes. It will then create a map (Key - > Unique value of index, Value -> row_ids of data where that value exists). Offload this map to the Index. Introduce row_ids in case there isn't a primary key in the table. 
		* According to binning decision (Which proxy is responsible for which table(s)), forward the primary-keys to the executor proxy to build its own BST (real keys).  
	2)  Query: `select * from table1 where age = VAL`
		* `FetchValues(Name=tableName,Index=IndexName,IndexVal=VAL)` :
		* Index proxy will traverse its BST (on specified Index), it will issue a batch to the index. It will decrypt and forward the value (List of doc_ids) to the executor proxy responsible for the doc_id range of this table. 
		* Executor Proxy creates its own batch and executes it on the server. It returns result to index proxy. (Without $F_r$ )
		* Doesn't work if selecting value where no index exists. Have to stream full table and filter value out that way. 
	3)  Query: `select * from table1`
		* `FetchTable(Name=tableName)`
		* Index proxy traverse BST on any of the index created for table 1 (Whichever is smaller). Issues request to Index Waffle, Decrypts Doc_ids and forwards the doc_ids to their designated executor proxies. 
		* Executor Proxy runs the batch, back propagates all fetched results to the index proxy. (Without $F_r$).
	4) Query: `select col from table1`:
		* `FetchColumn(Name=tableName, Col=ColName)`
		* Regardless of the col having an index or not, We will need to fetch the entire table and filter out column at the proxy.  
	5) WriteBack Phase: 
		* The executor proxy is responsible for writing the doc_ids with new timestamps back to their _**new**_ partition. The decision to evict is taken by the executor proxy. Hence before the executor proxy returns the data to the index proxy, it assigns a random partition to each key (or set of keys) that it will evict to. This is done so that the index can save the new location in the index. 
	6) Example:
		1) Client calls `fetchValues(Employees,Age,20)`, get me all employees who are 20 years old. 
		2) Index will traverse age bst and generate `get(PRF(table|age|20|TmeStp))` 
		3) Proxy will get value and decrypt it to `[(doc_id1|P1),(doc_id2|P2),(doc_id3|P1)]`
		4) This list is forwarded to executor proxy responsible for these doc_ids. (Hash doc_ids and find executor proxy)
		5) Executor Proxy runs the read batch, it returns to the index `(doc_id1|P2)|Value`
		6) Index proxy will take this new key and write it back to the index. 
		7) Here while the read is taking place, the index has to resides at the index and cannot be written back until the new key is received from the executor proxy. 
		8) We can optimize this maybe by delegating the task of new partition assignment to the index. This way executor knows the next partition to write the key to, instead of assigning it itself. 
		9) Idea: Hashing the doc_id to find which proxy to assign it to is dependent on the quality of a hash function. What if we assign linear primary keys to the **entire database** across all tables. Pkey range is then 1-N. Split range between proxies? 
	7) Dummy Values:
		* Each waffle (Index Proxy and Each Executor Proxy) will be initialized with some dummy values. (Dummy values are fixed, so final partition size is C+D) where C is the number of real keys and D is the number of dummy values. 
		* Few option questions:
			1) If we assign an executor proxy to one partition, In the read phase, the real-key bst, the timestamp would need to updated in all executor proxies (since each executor can potentially run a query for each value within a given time period). Dummy BST is independent for each partition. 
			2) If we don't assign an executor proxy to one partition, in the normal read phase, BST for real keys would need to be synced across all executor proxies. We will also need to sync in the read phase. (So if the doc_id comes to any proxy, it has the latest timestamp). Dummy BST will also need to be synced, since any proxy can access any partition. 
			3) If we assign the doc_id to executor proxies, the real-key bst would not need to be synced. But the dummy BST for each partition would need to be synced across all executor proxies. Since each executor can access a different partition each time, it should have the updated dummy bst of that partition to query dummy objects. More fine-grained syncing needed since we don't want an executor proxy to access dummy value twice. There needs to be some communicated between what dummy has been read or not.
	8) Possible Solution:
![[Pasted image 20240215145648.png]]
The dummy space is split between execution proxies. Each executor has its own "Space" within the dummy keys present at each partition. This way, if we use option (3), where doc_ids are mapped to a specific executor, it can batch the dummy keys it owns on that partition with the real keys avoiding any synchronization. But each executor would not need to hold P dummy bsts where P is the number of partitions. 

One way to go around this is to create dummy keys in a way that we can access all of them in a way without using a bst. 
Each executor proxy can hold some metadata like 
```json 
{
	"Partition_One_Dummy_Read":{
		"Total":100,
		"Key":'cscs',
		"Access":20
	},
	"Partition_Two_Dummy_Read":{
		"Total":70,
		"Key":'xyz',
		"Access":50
	},
	"Partition_Three_Dummy_Read":{
		"Total":120,
		"Key":'abcd',
		"Access":80
	}
}
```

For generating batch with 20 dummy keys to Partition1, executor one can: 
Access `Partition_One_Dummy` and generate keys as KEY|ProxyID|Access
where access would range from 20 - 40. For Example `cscs|1|25`. Once `Access` has reached `Total`, a new key is generated and values from old_key are given to the deletion thread. The server still won't be able to distinguish Real from Dummy. 

For writing back, we can have an identical metadata json. 

```json 
{
	"Partition_One_Dummy_Write":{
		"Total":100,
		"Key":'cssc',
		"Written":20
	},
	"Partition_Two_Dummy_Write":{
		"Total":70,
		"Key":'xsssyz',
		"Written":50
	},
	"Partition_Three_Dummy_Write":{
		"Total":120,
		"Key":'abaacd',
		"Written":80
	}
}
```

So each time you read a different dummy value and write a different dummy value. Given that we will read and then write the same # of dummy values, we shouldn't have more than two keys that are being used simultaneously (I might be wrong.)


##### Range Queries 

1) Initialization: `CreateTable(Name=tableName,Data=data,Index=[Col1,Col2],Range=[Col1])`
	* Uses the same function as the first with optional argument of Range. It takes in a list of columns on which the range index needs to be created. The col must also exist in the Index list passed to the function. 
	* For the column that is specified in the range argument. The function calculates the range (Min and Max), which is the domain of the index. It then creates a segment tree on the this domain. This segment tree will act as a BST for this index at the index proxy. The mapping of K/V pair is then offloaded to the index waffle: 
		* (KEY|Value) = `(10-20| doc_id1||p1,doc_id2||p2,doc_id3||p1)`. `10-20` is the node label of the segment tree. 
2) Query: `Select * from table1 where age between 10 and 20`
	* `FetchRange(TableName,RangeIndex,Min,Max)`
	* This function will first traverse the segment tree at the index proxy. In case a the range doesn't have valid doc_ids (Timestamp will be NULL), return to the cclient and empty list. In case there is a timestamp, issue a query to the index waffle. 
	* Decrypt Doc_ids and forward the requests to each executor proxy. 
	* Executor proxy delegates the results back and index proxy replies to client.
	* Updates external index. 
3) Query: `Select grade from table1 where age between 10 and 20`
	* Runs similar to (2), instead of returning everything, the index server (or executor server) will filter out grade. 
4) Query: `Select * from table1 where age>10`
	* Again, similar to 2, but in this case the entire subtree of the segment tree will be traversed to find all non-empty (no null timestamp) nodes that satisfy the condition. 


##### Joins: 

1) Initialization: 
	* This would be in two parts: 
		1) `CreateTable(Name=tableName,Data=data,Index=[Col1,Col2])`
		2) `CreateJoin(Table1,Table2,Table1Index,Table2Index,IndexExists=T)`
	* In the case where both tables are joined on an indexed column (we have a BST at the index proxy), we can traverse both trees, collect all possible unique values and then take out the intersection of them. This gives us the key value existing in both tables.
	* In the case where we join on columns that are not indexes: 
		* `CreateJoin(Table1,Table2,Table1Index,Table2Index,IndexExists=F)`
		* The program would have to create a join map and store it locally. This can be a list of json with each object key being the unique value of join and value being the keys in each table.
		* `[{'10':{t1:[doc_id1||p1,doc_id2||p2],t2:[doc_id5||p1,doc_id6||p2]}}]`
2) Query: `Select * from t1 join t2 on t1.age = t2.age`
	1) In the case where age does have an index in both tables, we will traverse both BSTS, collecting unique ids and then calculate the intersection. We then issue queries to the index waffle for both. 
	2) In case we don't have an index, we scan the entire join map and forward doc_ids to the executor proxies. In case the join map is offloaded, we will need to then fetch it back (The join map will need its own BST)
3) Query: `Select * from t1 join t2 on t1.age=t2.age where age = 20`
	* In case an index exists for age, traverse bst of each table and issue requests to index waffle. In case where one table doesn't have value, return empty list to client without issuing any request to index waffle. 
	* In case we have a map, find the key in the map and forward its value to the executor servers. 

##### Additional Complexity with Integrated Index:
* Index proxy will have to follow a similar dummy logic like executors
* Index will need to house-keep where each key resides (which partition).


*TODO*

***Haven't covered range joins**
**Haven't Covered Aggregates**
**Haven't Covered Updates**


##### Tasks: 

$d/f_d$ = $N-C/B+F_R$ --> Basically see how to calculate d with the above technique, 

Aggregate batch from each executor into one large so server sees one batch together. 

Partition Alpha (Server sees Alpha, each proxy issues Alpha/3 in case we have 3 executors. )

Epoch Based

Issue: One proxy runs faster than the other and then we cannot give a bound to the **server** about alpha. Even though executor pulls k1 after every 30 (Alpha), the server sees that its being accessed after 300 server accessess. 

Collect B/3 batches and then send B (Single. )

Distributed Proxy system 

Index is separate and then think about load balancing the system. 

Underlying problem: When we partition state across proxies, we don't want one to make 60 accesses (while still maintaing alpha/3) while other proxy is still lagging behind. 



## Feb 23 

### Coordinator Node
#### Case 1: Independent Indexing (Separate Waffle)
![[Pasted image 20240222223432.png]]

**Issue**: In the previous design, the executor nodes could out-run (out-pace) each other. This can cause issues with the $\alpha$ guarantees we want to provide. For example, if executor three is on its third batch of queries and executor one is still collecting requests for its 1st batch, then overall the server has seen 3 accesses. Executor one only guarantees alpha from its own perspective (I guarantee that I will access each object within $\alpha$ accesses **I** make).

**Potential Solution**: 

1) Introduce a coordinator node that "collects" queries for each executor node. It then issues these queries to each executor together. Basically it "batches" the queries before they are sent to be executed. 
2) The coordinator will maintain a queue per executor node of size $B - F_d-F_r$
	* B is the batch size set for the executor. 
	* Fake queries on Dummy objects will be added by the executor itself. Fake queries on real objects will also be added by the executor as well. 
	* Thus coordinator checks: `SizeOfEachQueue >= R`. It will then dequeue $R$ requests and send it to corresponding executor. 

3) Steps for Processing a Request: 
	1) Clients send request to Coordinator.
	2) Coordinator Asks Index to provide doc_ids for requested items.(Using Indexes or Join Maps).
	3) Coordinator assigns key to its designated queue (Hash(doc_id)). 
	4) Checks if all Queues are filled to $B-F_d-F_r$. If so, Dequeue and send doc_ids to executor. 
	5) Executor fetches documents and returns the values to the coordinator. 


#### Case 2: Integrated Indexing (Index stored on the partitions)


![[Pasted image 20240222232706.png]]

In this approach, the "index" is a separate process that is responsible for maintaining the Point and Range BSTs for indexes columns (across all tables) along with Join Maps. We can think of this as the Index oracle that provides the information needed to access the index values. For example:

1) Select * from Table1 where Age = 10;
	* Coordinator calls `getPoint(Table=Table1,Col=Age,Value=10)`, the interface exposed by the Index process. 
	* The Index process in this case returns `KEY|PARTITION'. 
	* In this case, partition information would need to be stored for the index value as well. 
	* The Coordinator will then call `Hash(key)` and insert it in that executors queue. 
	* Executor fetches values Coordinator adds back again to each executors queue. 
2) Select * from Table2 where Age Between 10 and 20; 
	* Coordinator calls `getRange(Table=Table2,Col=Age,MIN=10,MAX=20)`
	*  Index returns a list: `[(10|PARTITION),(11|PARTITION)...]`
	*  Same process follows as above. 

In this case, the binning process (Assigning Which executor proxy handles which keys) needs to be come after creating indexes. So we have the total key space (index Keys and Data Keys) to map to executor proxies. 


Deleting Items? 

1) Case where we delete all values of Age=10. 
	* Index process can delete the key out of its bst (mark it for example).
	* The key still resides in the executor proxy BST. For example key is: table1_age_10. 
	* Can we introduces a background tasks that deletes these nodes from BST?



Open Questions: 

1) How much of a bottleneck is the coordinator? 
	1) What if popular key resides only on Executor 3? It is likely cached but we can't access it until rest of the queues fill up. Will this be a large bottleneck? Can incorrect mapping or distribution lead to starvation? The integrated index might help with this problem since the index of a popular item lies on a different partition (hence executor). 
2) Should our partitioning be probabilistic or deterministic? The issue(?) I could think of with probabilistic approach is that we can't control how keys map to partitions. Lets say: 
	* Randomly all keys that Executor 1 got were in Partition 1. 
	* Executor 1 wouldn't need to access Partition 2 or 3. Where as Partition 2 and 3 might access Partition 1. 
	* If we think of the outsourced space a physically one unit, then it still looks like 3 machines accessing a single data server. The logical separation exists in the key spaces the executors access. 
	* But if we have physically distinct machines, then partition 1 got (3) access from distinct machines where as partition 2 and 3 got (2) accesses. Doesn't this affect Alpha for partition 1? (Proxy guaranteed Alpha < Alpha Server Sees). Can a deterministic approach to partitioning keys help (all partitions accessed each time).  
3) A possible solution for the bottleneck is to have timeouts.
	* Instead of adding Dummy keys, we let the executor know of the timeout and the executor fills in the batch with $F_r$. This way our $\alpha$ for that batch would be lower than the upper bound set by the default parameters. 
	* This can potentially cause a difference between the $\alpha$ histograms of the server accesses. The server coupled with an executor experiencing more timeouts would have a less tail heavy distribution, where as a normal executor would have a tail heavy distribution. 
	* The difference in these graphs/across partitions can leak information about where popular keys reside? 
	* Trade off: Improve performance, but leak more information? 

* Think of keyspace as a since large keyspace. 
* Each executor guarantees $\alpha'$  where as server sees $\alpha =3\alpha'$  where 3 is the number of partitions/executor nodes. 
* Batching over time intervals/epochs, just duplicate keys until it reaches R
* Coordinator needs to be as dumb as possible
* Integrated Index
* 1-N Index for recreating.


## March 1

#### Arx Index Planning

Assumption: We have input a set of query patterns. 
Index planning also needs to know indices already built (regular indices) to guarantee same asymptotic performance. 

Needed because no direct map from regular indices --> Arx Indices 
Arx's Indices introduce constraints:
	1) A regular Index can server for both range and equality queries. Not true for Arx. Different index for different operations. 
	2) ArxEQ index on (a,b) cannot be used to compute equality on just a. It performs a complete match. 
		1) Map --> Distinct values of Age to Count. So here it would be distinct pairs of (a,b) --> Count. 
	3) A range or order by limit on a sensitive field can only be computed using an ArxRange index. It can no longer be computer after applying a separate index. I don't know why? 

Admin can also explicitly specify certain fields to be non-sensitive and also declare compound indices on mix of sensitive and non-sensitive fields. Queries can have sensitive and non-sensitive fields in the where clause. 

Example, admin says build index on a. 
Does query: `where a = and s>` where s is sensitive. 

Admin expects the db will first filter by a and then by s. 

In naive solution, the system could create a ArxRange on s alone, then the db will first filter by S and then filter by A. Index on A would be then useless. It needs to run ArxRange on S because its a constraint by the system that range queries need to be done by ArxRange. I think you can't perform it after applying another index because that might leak information? I'm not sure. 

In order to improve performance, Arx builds a composite index ArxRange on (a,s)

The planner basically Zips the indexes so that they perform similarly to the regular db indices. 

Not much help in our case.  We don't require query patterns. Just what indices to make. 



#### SQL mapping 

For the integrates Index: 
MetaData present: Table Names, Table Columns, Index Columns,  Join Maps

PK is assigned from 1 - N. Keep record of PK range where a table resides (T1 resides from 20-100)

Initialization: The tables rows and columns are converted to Key/Value pairs. So if we have a table with N rows and X Columns, we will have: 
	1) $N*X$ key/value pairs. 
	2) Each Key would be of the form `Table_Name/Col/Pk`

For each point index we create, the # of records per index would be: 
	1) UNIQUE(Col_Values) 

For range index we create, the # of records per index would be:
	1) # of valid ranges (10-20, 20-30, 30-40) for each value
	2) n log (n) rows in case we want to issue 1 index fetch.
	3) # of valid ranges in case we want multiple index fetches. 

1) Point Queries:
	* `Select * from t1 where age = 10`
		* Index is created as Key = `table_name/col/val` so in this case it would be `t1/age/10`. The value for this would be the list of doc_ids/pk of t1 where age is 10. The coordinator simply constructs the key after doing a sanity check that an index exists. After receiving pk from index, the server can construct keys for the table `t1/<col>/pk` and issue these to the appropriate executor. 
	* `Select col1 from t1 where age=10`
		* Same as above, but instead of issuing request for every col, only done for age col. 
	* `Select * from t1`:
		* Using the PK range for t1, you can issue a query for all values. 

2) Ranger Queries: 
	- Also see if we can just convert to range query. 
	At index creation time, construct a segment tree (interval tree) and associate the primary keys of rows that fall into particular intervals. We can call this the label generation part where label is for example 10-20, or 35-40. 
	When a user gives us a range, lets say age between 15 and 20. We will query this segment tree to find a valid range covers the query range. In this case 10-20. Then the document ids are fetched for this range and the associated values are received. 
	* `Select * from t1 where age>10`:
		* First ask segment tree for valid ranges. Lets say it returns 1-4 and 5-10. 
		* Then issue execution for `t1/age_range_index/1_4` and `t1/age_range_index/1_5`
		* Upon receiving doc_ids, we generate query `t1/<col_name>/pk`
	* `Select col2 from t1 where 20<age<40`
		* Same thing as above, just only issue requests for col2 instead of every column. 
	* `Select col2 from t1 where age>20 and salary=40`
		* Assuming we have an index on salary and age.
		* Issue a request for both indexes, range and salary. 
		* Upon receiving both values, calculate the intersection. If doc_ids exist then fetch values, otherwise return null. 
		* Make sure both are in same batch to get responses together?
3) Join Queries: 
	Construct join maps on the attributes of that are to be joined. 
	* PK-FK join:
		* A simple list of PK-->FK join. This is for a complete join `select * from t1 join t2 on t1.pk=t2.fk`
		* For example if pk in t1 is disease and fk in t2 is symptom. 
		
		T1 
		
		| ID | Disease | Meds | Stay   |
		|----|---------|------|--------|
		| X  | Flu     | abc  | 2 days |
		| Y  | Fever   | xyz  | 1 day  |
		
		T2
		
		| ID  | Name   | Age | Symptom |
		| --- | ------ | --- | ------- |
		| 1a  | Haseeb | 24  | Flu     |
		| 2b  | Ahmed  | 25  | Fever   |
		
		Map would look like: 
		
		| PK | FK       |
		|----|----------|
		| X  | 1a,3a,4a |
		| Y  | 2b,1b,2d |
		
		Generating queries would mean simply iterating over the map and issuing queries: 
		`t1/<col>/X` and `t2/<col>/1a` etc

	* Join on Indexes column: `select * from t1 join t2 where t1.age=t2.age`
	* Similar sort of map as shown above. We would like to also keep a record of the values of age in this case in the map so we can support additional filtering like `select * from t1 join t2 where t1.age=t2.age and age=20` or `select * from t1 join t2 where t1.age=t2.age and age= between 10 and 20`

4) Updates:
	* `Update t1 set salary = 20 where age = 20`
		* Fetch pk from index where age = 20. `t1/age_index/20`
		* Insert into cache the K:V pairs `t1/salary/pk : 20`.
		* Issue a read request for old values from system. Discard them (Don't insert them into cache)
		* Requires executor to hold more state (Know what to not add to cache or what to add)
	* `Update t2 set salary = 40`
		* Similar to the past part. But here we would need to do a full table scan to get all primary keys.
	* `Update t1 set age=20 where name='haseeb'`
		* In this case we don't have an index on name. We will need to pull all records
		* One we find the pk with the name haseeb, we will update its age column by adding it to cache and issuing a read (That shouldn't save to cache)
		* Will also need to update index. 

5) Aggregates:
	* MIN/MAX: Trivial to keep this as metadata on proxy for each numeric attribute. Verify whenever an update happens. 
	* COUNT: 
		* `select count(*) where age>10`
			* Issue a request for age>10 to its index. Count distinct pks returned.
		* `select count(*) from t1`
			* Return table pk range count 
		* `select count(col2) from t1 where col2=10`
			* In case col2 doesn't have an index, issue `t1/col2/pk` for each pk of the table. Then filter out for 10. 
	* Distinct: 
		* I think its easier to keep as metadata on the proxy. Verify if it changes on updates
	* SUM: 
		* `select SUM(salary) from t1 where age>10`
			* Fetch doc_ids/pk for rows>10.
			* Issue requests for salary using those pk. Sum them

As long as you are doing an aggregate with an indexed attribute to filter values, we can do it. If not then a full table scan is needed. 



#### Pseudocode
[[updatedFlow.canvas|updatedFlow]]
1) Parser : Handles parsing sql into something the indexer understands
2) Indexer: The brains of the operation. Does all house-keeping for queries, optimizations etc. 
3) Load-Balancer: Makes sure no one proxy runs ahead of another
4) Executor proxy: Follow $\alpha - \beta$ uniformity to provider protection against access pattern and volume attacks. 


Global Variable: PK <-- 0 
##### Table Creation

`CreateTable(TableName,TableData,IndexColName,IndexTypes)`

For example `CreateTable(Employee,<csv_file_path>,[age,salary],[point,range]`

Operations:
1) Read the tableData into memory and assign each row an incrementing PK.  
2) For each index in the IndexColName:
	1) Identify what type of index is it (using IndexTypes)
	2) If it is a point index:
		1) Identify the Unique values for that column `SET(tableData[age]` for example. 
		2) Create an map, initialize it with keys: `Employee/age/val1` .. `Employee/Age/valN`
		3) Iterate over the entire table. Concatenate the PK of that row to the value for the assigned key. (If row has age 20 and pk 12, then go to `Employee/age/12` in the map and add 12 to its value)
		4) Save this map.
	3) If it is a range index:
		1) Identify the Min and Max from the data.
		2) Call `createSegmentTree(min,max)`
		3) Initialize an empty map. 
		4) Iterate over entire table. For each value, call `segmentTree.getLabel(val)`. This would return the ranges this value maps to for example (2-8,4-5).  Insert the PK of this row to its associate keys in the temporary map. For example if salary is 5, then add pk to keys `employee/salary/2_8` and `employee/salary/4_5`. We can also
		5) Save this map.
		6) **This method creates log(n) labels for each value. So a total of n log n k/v pairs. We can possible just fetch the last label (4-5) instead of labels higher up in the tree to reduce # of labels**. 
3) Generate key/value pairs for each row and each column:
	1) For row in tableData
		1) For col in tableData 
			1) key = `employee/col/row.pk`
			2) value = `tableData[row][col]` <-- 2d matrix indexing
			3) tableMap.key = value.
4) Save the tableMap and index maps. 
5) Once all tables have been processed. We can proceed with binning process over the entire dataset (data + indexes)

##### Indexer:
Responsibilities: 
1) Parser provides high-level requirements (give me colx where age is between x and y). 
2) Indexer takes these inputs and generates the appropriate keys to index using metadata information, depending on availability of indexes etc. 
3) It does housekeeping for all requests that are being indexes and waiting for values, requests that have received their values and need to be returned. 
4) It also handles optimizations for aggregation.
5) Will also handle logic for updates. 



Structure of relational data
and query patterns

TPC-C 
Berkley Cloud Benchmark 
Electronic Health thingy 
Arx


Pseduocode for one type of query End-To-End


## March 8th + March 15th 

##### Difference between tpc-c and tpc-h:

1) Kaggle
2) UCI --> Their dataset collections. 
3) Chicago Crime Dataset. See if they use SQL queries. 


Stick with TPC-C. Don't do TPC-H (Analyitcal.)
https://web.archive.org/web/20120919183401/http://www.tpc.org/information/benchmarks.asp

Group-by, Order-by --> See if we can do in future 
(Keep Both)

TPC-C : Does Removing transactions remove moving out of the queries? (Correctness.) --> See if TPC-E is non-transactional. Maybe we can use that. 
TPC-C simulates a complete computing environment where a population of users executes transactions against a database. The benchmark is centered around the principal activities (transactions) of an order-entry environment. (No dbgen on official tpc website.)

TPC-H:
The TPC Benchmark™H (TPC-H) is a decision support benchmark. It consists of a suite of business oriented ad-hoc queries and concurrent data modifications. The queries and the data populating the database have been chosen to have broad industry-wide relevance. This benchmark illustrates decision support systems that examine large volumes of data, execute queries with a high degree of complexity, and give answers to critical business questions.

Information on Schema + Queries: 
https://github.com/cmu-db/benchbase/blob/main/src/main/resources/benchmarks/tpch/ddl-cockroachdb.sql

TPC-H has a dbgen utility which creates a database. We also have a list of queries and corresponding answers to those queries. 


##### Berkley Cloud Benchmark (Neutral)
https://amplab.cs.berkeley.edu/benchmark/

Has Queries + Workload. 


##### Medical Health Records:

**1) ORBDA: A dataset for benchmarking OpenEHR systems.** 

This project aims to assemble and make publicly available a database generated in accordance with the specifications of the openEHR foundation to be used in studies evaluating persistence mechanisms for systems based on openEHR. A set of Electronic Health Records (EHR), based on the openEHR reference model, can be generated from publicly available data to simulate a production EHR system. The data consists of APACs and AIHs made publicly available by DATASUS, without patient identification, from 2008 to 2012. Archetypes and templates were created to convert data from datasus .dbf files into pseudo RES, generated according to the specifications of the openEHR foundation.


https://www.lampada.uerj.br/projetos/orbda/

https://www.ncbi.nlm.nih.gov/pmc/articles/PMC5749730/
https://github.com/pradeepsen99/Patient-Similarity-Prediction/blob/master/DataBaseScripts/orbdaCreateTables.sql

Have Schema + data, No Queries (we can create our own?)
Comments: I think we can test range with conversion to point queries first. Then decide if we want to implement tree based range or not. 


**2) Medical Information Mart for Intensive Care III** 

A large, freely available database comprising de-identified health-related data associated with over forty thousand patients who stayed in critical care units of the Beth Israel Deaconess Medical Center in Boston, Massachusetts.

https://github.com/MIT-LCP/mimic-code/

Has data + some exploratory queries. 

**3) eICU Collaborative Research Database**
Similar to MIMIC-III, the eICU database has a GitHub repository where users have contributed SQL code for various analyses.
https://github.com/MIT-LCP/eicu-code


Some of the healthcare datasets we saw in papers (for example FreeHealth, isn't available online? The website doesn't work anymore) 
Generally, datasets are available but we would need to create queries ourselves. 



<hr>

##### Pseudocode

* PK Per table (So we can do inserts. Otherwise inserts would cause overlapping pks table1 ends at 99 and table 2 starts at 100)
###### Database Initialization: 
Initialize 
$R \gets$ Whatever we set it to. (# of Real Values in a batch) 
$plainTextKV \gets \emptyset$ //Total set of all k/v pairs generated at init. Before binning. 
$metaData \gets \emptyset$  (min,max,pk_start,pk_end,column_info) 
$pointIndex \gets \emptyset$ 
$joinMaps \gets \emptyset$

<hr> 

DBCaller --> Calls CreateTable for each table --> Does Bucketing --> initializes each Waffle. 
#### 1) Initialization:
`CreateTable(tableName,tableData,indexColName,indexTypes)`

For example `CreateTable(employee,<csv_file_path>,[age,salary],[point,range]`

**Assumption: Treating Ranges as a list of points. In case we maintain a tree, this part has to change.** 

Operations: 
$data \gets tableData$ //ReaDatata
$pointIndexList \gets \emptyset$ //List of indexes. Store indexes here.
$allColumns \gets data.GetColumnNames()$ 
$pk \gets 0$
$for$ $(i,index)$ $in$ $indexTypes.enumerate()$: // Initialize the indexes
	$colName \gets$ $indexColName[i]$ 
	$uniqueValues \gets$ $Set(data[colName])$ //Set returns unique values in list.
	$colNameMap \gets$ Initialize Map. Use $uniqueValues$ as Keys, Value is $currentRowVal$ which is $\emptyset$.  		
	$pointIndexList.push(colNameMap)$

$for$ $row$ $in$ $data$:
	$for$ $col$ in $allColumns$: 
		$key \gets$ $tableName/col/pk$ 
		$value \gets row[col]$
		$for$ $index$ $in$ $pointIndexList$:  //Index creation. For each row, add Pk to its corresponding index.
			$if \ col \ in \ pointIndexList:$ 
				 $index[tableName][col].push(PK)$ 
		$outSourceKV.push((key,value))$ //Add to total K/V pairs generated. 
	$pk++$

$for$ $index$ $in$ $pointIndexList$.: 
// Replace the for loop below with Jolines indexing function/logic.  It will return a set of key/value pairs and metadata
	for $(key,value)$ $in$ $index.items()$:
		$indexKey \gets tableName/index/key$ 
		$outSourceKV.push((indexKey,value))$ //Add to total K/V pairs generated. 

$tableInformation \gets (min,max,0,pk-1,allColumns)$ //$PK-1$ is last row_id. Min/Max for each column. 
$metadata.push(tableInformation||index\_information)$ 
$pointIndex.append(pointIndexList.indexNames())$ //Do as a Map --> One index metadata per table. 

<hr> 

`CreateJoinMap(table1,table1Column,table2,table2Column)`
For example `CreateMap(Employee,age,Salary,age)`

//Parallel Computing --> Create Tables/Joins in parallel --> Save everything to file. 

Operations: 
$joinedTable \gets sortMergeJoin(table1,table1Column,table2,table2Column)$ 
//We can also use other join algorithms like hash join. 

$mapID \gets$Unique identifier for this join. 
$joinMap\gets$ empty map. 
$for$ $row$ in $joinedTable$:
	$key \gets row.table1Column$
	$value \gets row.table1PK$
	$value2 \gets row.table2PK$
	$joinMap[key].Pk1.append(value)$
	$joinMap[key].Pk2.append(value2)$
	`{'key':'pk1':[],'pk2':[]} --> {'20':'pk1':[1,3,2,5],'pk2':[4,2,5,1]}`
	
$joinMaps\_mapID \gets joinMap$
$joinMaps.push(joinMaps\_mapID)$ //Add to join Maps 

<hr> 

###### 2) Parsing 

`ExecuteQuery(statement)`

For example `ExecuteQuery('select age from Employee')` 

No direct pseudocode. 
In the parsing module, we want to only parse the SQL statement into a Abstract Syntax Tree and call appropriate functions of the Indexer/Query Resolver. 

<hr> 

#### 3) Query Resolver 

//Use boolean --> Use Range Index --> True (Fetch and use Index instead of point queries)
`doSelect(tableName,colList,searchColumn,searchValue)` //Works for range as well. searchValue would just be a list (min,max)
For example: 
`select * from table1` --> `doSelect(table1,all,NULL,NULL)`
`select * from table1 where age=10` --> `doSelect(table1,all,age,10)`
`select salary,name from table1 where age=20`--> `doSelect(table1,[salary,name],age,20)`
`select name from table1 where age between 10 and 20`--> `doSelect(talbe1,[name],age,[10,20])`


Operations:
$if$ $searchColumn$ $not$ $in$ $pointIndex$ $or$ $searchColumn$ $is$ $null$:
	$return$ $fullTableScan(tableName,colList,searchColumn,searchValue)$
$else$ 
	$indexName \gets pointIndex.getIndex(searchColumn)$ 
	$indexKey \gets indexName/searchValue$ //When searchValue is a list, indexKey will be a list for each key in range. 
	$doc\_ids \gets waffleAddBatch(indexKey)$ 
	$while$ $(doc_ids.finished = false)$  {
		//Busy Wait. 
	}
$requestBatch \gets \emptyset$
$if \ colList=all:$
	$columnList \gets metaData(tableName).columnNames$
$else:$
	$columnList \gets colList$
$for \ id \ in \ doc\_ids:$ 
	 $for \ col \ in \ colList$
		 $key \gets tableName/col/id$  -- Assuming id contains partition information. 
		 $requestBatch.append(key)$

$return \ waffleAddBatch(requestBatch)$

<hr> 

`doJoin(table1Name,table2Name,colList,searchCol,searchValue,joinColumns`
For example:
`select * from table1 join table2 on t1.id=t2.e_id` --> `doJoin(table1,table2,all,NULL,NULL,[id,e_id])`
`select name from table1 join table2 on t1.age=t2.age where age=10` --> `doJoin(table1,table2,[name],age,10,[age,age])`
`select name from table1 join table2 on t1.age=t2.age where salary=200` -->`doJoin(table1,table2,[name],salary,200,[age,age])

Go from best to worest case. --ToDo. 
Operations:
$if$ $joinColumns$ $not$ $in$ $joinMaps$: //Check if joinMap exists
		Improve performance of this (streaming data --> parallel fetching of t1 and t2). 
	$tableOne \gets fullTableScan(tableName,colList,searchColumn,searchValue)$ //Get Table 1
	$tableTwo \gets fullTableScan(tableName,colList,searchColumn,searchValue)$//Get Table 2
	$return \ tableJoin(tableOne,tableTwo,joinColumn,searchCol,searchValue)$
$else$ 
	$requestBatch \gets \emptyset$
	$postProcessing \gets false$ 
	$requiredJoinMap \gets joinMaps[joinColumns]$ //Find appropriate joinMap
	$if \ requiredJoinMap \ includes \ searchValue:$
		$keys \gets constructRequestKeys(requiredJoinMap,searchCol,searchValue)$  //The search value was recorded when we made the map (Example 2)
		$requestBatch.append(keys)$ 
	$else:$ 
		$postProcessing \gets true$
		$keys \gets constructRequestKeys(requiredJoinMap,NULL,NULL)$  //Do a normal join, containing all possible values for searchCol. (Example 3)
		$requestBatch.append(keys)$ 
	
	$data = waffleAddBatch(requestBatch)$
	$if \ postProcessing = true:$
		return $filteredData(data,searchCol,searchValue)$ (example 3)
	$else:$
		$return \ data$ (example 2)

<hr> 

`doAggregate(tableName,aggregate,colList,searchCol,searchVal)`
* `select count(id) from table1`  --> `doAggregate(table1,count,id,null,null)`
* `select sum(age) from table1` --> `doAggregate(table1,sum,age,null,null)`
* `select min(age) from table1`--> `doAggregate(table1,min,age,null,null)`
* `select sum(salary) from table1 where age between 20 and 30` --> `doAggregate(table1,sum,salary,age,[20,30])`

$tableMetaData \gets metadata.get(tableName)$ Returns metadata for a particular table. 
$if \ aggregate = min \ or \ max:$
	$colMetaData \gets tableMetaData[colList]$ --> 
	$return \ colMetaData[aggregate]$ --> returns desired aggregate.
$if \ aggregate = count$ --> fix this, not working for last example count* --> if no where clause then return first if. 
	$if \ colList == all:$
		$return  \ tableMetaData.pkRange()$ --> Won't work for deletes.--> maintain another delete counter. total = pk_range - delete. 
	$else \ if \ searchCol \ has \ index:$
		$return \ count(doSelect(tableName,colList,searchCol,searchVal))$ -->Do a selection and then count. Optimize by calling an index only function here.
	$else:$
		Do a full table scan and check where the conditions are true. --> Maintain a map of unique values for benchmarking. Also do for worest case.  
$if \ aggregate=sum:$
		$return \ sum(doSelect(tableName,colList,searchCol,searchVal))$ --> Do a selection and then sum. 
$if \ aggregate = average:$
	//Recursively call the aggregate function for count and sum. 
	$count \gets doAggregate(tableName,count,colList,searchCol,searchVal)$
	$sum \gets doAggregate(tableName,sum,colList,searchCol,searchVal)$
	$return \ sum\div count$ 

Either:
1) Consult metadata to answer query
2) call `doSelect` or `doJoin` and aggregate the result. 

<hr> 

`doUpdate(tableName,colList,newValue,searchColumn,searchValue)`
* `update table1 set age=20` --> `doUpdate(table1,age,20,null,null)`
* `update table1 set age = 20, salary = 10 ` --> `doUpdate(table1,[age,salary],[20,10],null,null)`
* `update table1 set salary = 20 where age=30` --> `doUpdate(table1,salary,20,age,30)`

Operations:
$requestBatch \gets \emptyset$
$if \ searchColumn == NULL:$ //Full column is being changed
	$for \ col \ in \ colList:$
		$if \ col \ has \ index:$
			$pk\_tableRange \gets metadata.getPkRange(tableName)$
			 $for \ doc\_id \ in \ documents:$
				 $key \gets tableName/col/doc\_id$
				 $value \gets newValue$
				 $requestBatch.append((key,value,op=write))$
			$newIndex \gets makeIndex(pk\_tableRange,age))$ //Jolines Function gets called here and returns a list of K:V pairs. 
			$for \ key,value \ in \ newIndex:$
				$requestBatch.append((key,value,op=write))$
			$deleteIndex(tableName,age)$ --> Call to delete old indexes on age (this would be multi-round). Call this before sending request batch to Waffle so delete operation is in front of the queue. 
		$else:$
			$pk\_tableRange \gets metadata.getPkRange(tableName)$ --> Move up outside of if/selse block. 
			 $for \ doc\_id \ in \ documents:$
				 $key \gets tableName/col/doc_id$
				 $value \gets newValue$
				 $requestBatch.append((key,value,op=write))$
$else:$
	$for \ col \ in \ colList:$
		$if \ col \ has \ index:$ 
			$documents \gets doSelect(tableName,col,searchColumn,searchValue)$ --returns truncated column depending on searchColumn
			$old\_values \gets \emptyset$ 
			$for \ doc \ in \ documents:$
				$doc\_id \gets doc.doc\_id$
				$key \gets tableName/col/doc\_id$
				$value \gets newValue$
				$old\_values.append(doc.value)$ -->Save old value to modify its index.
				$requestBatch.append((key,value,op=write))$	
			$removePkFromIndex(col,old\_values,documents.doc\_ids())$ --> Call function to fetch indexes of old_values and remove the given doc_ids from it. 
			$addPkToIndex(col,value,documents.doc\_ids())$ --> Calls function to add document_ids to the new values index. 
		$else:$
			 $documents \gets doSelect(tableName,col,searchColumn,searchValue)$ 
			 $for \ doc \ in \ documents:$
				$doc\_id \gets doc.doc\_id$
				$key \gets tableName/col/doc\_id$
				$value \gets newValue$
				$old\_values.append(doc.value)$ -->Save old value to modify its index.
				$requestBatch.append((key,value,op=write))$	

$waffleAddBatch(requestBatch)$
$return$ 

1) Generally, this would be the same as reading. The op in this case would be write, and the value would be the value passed by the user and not NULL. 
2) Ideally the existing waffle interface would handle the write, not the systems above it. 
3) `removePkFromIndex` and `addPkToIndex` are going to be expensive (multi-round). ToDo: Can we do any better? 

<hr> 

#### 4) Load Balancer:

Initialization: 
1) Initialize queues for each executor proxy. Access via $executorQueues$ variable.  
2) Initialize $E \gets NumberOfExecutors$ 
3) Start a thread that runs the function `executeBatch`
<hr> 
**Interfaces:** 

`WaffleAddBatch(keys)`

Operations: 
$for \ key \ in \ keys:$
	$hashValue \gets HASH(key)$ 
	$executorIndex \gets hashValue \mod E$
	$executorQueues[executorIndex].enqueue((key||value))$ 

<hr> 

`executeBatch()`

$while (true):$
	$for \ executor \ in \ executorQueues:$ //While Loop not for loop. 
		$if \ executor.size() != R:$ // R is real keys. A global constant 
			$continue$ 
	$for \ executor \ in \ executorQueues:$
		$batchKeys \gets executor.dequeue(R)$  //Dequeue R requests. 
		$executor.WaffleExecutor(batchKeys)$


<hr> 

#### 4) Executor:

`WaffleExecutor(Keys)`

1) Same interface given by current Waffle (`get_batch`). 
2) Have to modify so that:
	1) If a key does not exist in the waffle BST, then replace it with a **Fake Request on Real Key.**
	2) PUT Request --> TODO. Replace Dummy key with new value. 
	3) Be able to understand partition_id and ask the appropriate redis server.
		1) We could do this by maintaining a map between ipAddress --> Partition id. This is shared with all executors at initialization. 


<hr> 

##### Questions: 

1) Who should assign a new partition to the doc_id after fetch? 
	1) Should it be the executor itself, or the Indexer? 
	2) We need a strong proof that the keys are distributed randomly we in each execution, relatively similar # of keys get requested from each partition.
	3) Lets just assume redis is just one single keyspace --> Redis Cluster. 

2) Should we write a parser or convert queries to the functions manually for benchmarking? (already covered)
	1) Support TPC Queries
	2) Doesn't matter if your using parser or doing it pre-defined. But should we **correct**.

Query Optimization:
1) Do a filter first and then table join if join map doesn't exist. 
2) Search about query optimization.
3) Planner --> Can we use a plaintext planner and map it to our logic. 



https://www.querifylabs.com/blog/rule-based-query-optimization

1) Parser + Optimizer --> How can we do it in the least expensive way possible.
	1) I think biggest slowdown is 1round vs 2 round protocol. 
2) Can we use an optimizer from plaintext dbstores and map it to ours. 
	1) Maybe we can use a rule-based optimizer only. Cost based optimizers use analytics of the underlying data (histograms, distributions etc) to generate cost estimates. 
3) Plug-in Optimizer or Create our plan for every query we support. 
	1) For the purposes of benchmarking, maybe we can plan for every query ourselves? 



Index on age (employee table)

Key: employee/age/20
Val: 1||p1, 2||p1, 3||p2, 4||p3 ,5||p1 --> PK||Partition

What if we hash the timestamp to identify which partition its on? 
Index doesn't need to keep a track of partitions now. (For indexed and non indexed values)



Index updates: 

Updating index --> age/9/1 --> overflows to age/9/2 
This means we have N+1 now. One of the executors is now responsible for N/3 + 1 keys. This changes $\alpha$ for that executor proxy. Same thing for removing an index key. 


Think about reusing key/value 
Worest Case Partition --> 1M Rows all have same age. --> Don't do for project.
This is when we have two datasets, one has uniform dist of values for age and one has skewed. See difference in # of partitions made for each. 

The issue is that when we make changes to the index, its possible that an index gets removed (key moved from A to B, leaving A empty) or a new index partition gets created (table/age_index/10_2 --> table/age_index/10_3). This adds one more key to an executor. 

We can't have the same executor save this key either since the key `table/age_index/10_3` can get hashed to another executor proxy, so the indexing creation/deletion shouldn't be done by the executor. The query processor should delete/add new ones. 

Figure out how can we improve this. Can we just rely on the probability that over time each executor gets similar # of deletes and inserts? Can we give a window'd alpha bound? (Alpha is between some min max)? How can we deal with this? 

This is dependent on the workload as well. 




--> Fix changes in the code above. 
--> Move to latex with new changes 
--> Start re-implementing code for waffle and see if it works (Performance + Security Analysis.)
--> Go over the case given by florian. Do we have enough randomness? Case: Worst Case (We don't have a join map. Do full table scans.)

 
--> Keep small table in memory, stream larger table and write join to a WAL. 

--> Deduplication across batches (One batch reads and deletes, other is trying to fetch but sees NULL (Leakage))

https://redis.io/docs/latest/develop/use/pipelining/ 


--> Replace executor with a ORAM executor or a Pancake Executor. 
--> Each Executor gets same number of requests, that doesn't exactly leak distribution. 




## April 18 


Task: 
* Work on re-building waffle (Deadline end of April). If it takes too long then just use waffle as is. 
* Work on optimisations within waffle (parallel decryption, faster redis interface (mget,pipelines)). 
* Work on Updates (increase/decrease in keys for each waffle and how to handle alpha). 



Redis: 
https://medium.com/@jychen7/redis-get-pipeline-vs-mget-6e41aeaecef

Mget is faster than pipeline. Code already uses Mget.  --> I tested it, mget is ~1.5 times faster than pipeline. 

Alpha Calculation: 
![[Screenshot from 2024-04-18 20-58-49 1.png]]

Alpha = New timestamp - betaMap[key].  --> 

timestamp --> Indicates how many server accesses have happened (since its a counter that increments at the start of each batch)
betaMap[key] --> Refers to timestamp when it was written back to server. 

(Accesses Made) - (time when it was written) ==> How many accesses were made since last written. 
15 - 2 ==> 13 

15 --> Current Server access
2 --> When it was last written 

Now since your fetching on the 15th access, there have been 13 accesses where this key was on the server and not read from it. 



###### Simulation
![[Screenshot from 2024-04-19 00-30-53.png]]

I calculate Alpha, by incrementing N by 1 till 200, we see alpha grows as: 
![[Pasted image 20240419003135.png]]

Its a stepped function because of the ceil in the formula.

Idea: 
Instead of tinkering around with the batching parameters (B,$F_d$,R,$F_r$ ), we can adjust the cache by the same amount as N. 

If N goes from N --> N+1
C goes from C --> C+1 

![[Screenshot from 2024-04-19 00-39-36.png]]

Here I add i to both N and C. We get: 
![[Screenshot from 2024-04-19 00-40-31.png]]


If we test it on random values (Random Increases and decreases). I run 20,000 iterations: 

![[Screenshot from 2024-04-19 00-42-03.png]]
![[Screenshot from 2024-04-19 00-44-07.png]]

How C changes (% of N): 
![[Screenshot from 2024-04-19 00-48-07 1.png]]


###### Aggregates over time 
![[Screenshot from 2024-04-19 01-15-08.png]]

If changes are aggregated over time: 
![[Pasted image 20240419011550.png]]

C changes: 
![[Screenshot from 2024-04-19 01-16-09.png]]

##### in case we only change cache when there is an increment: 
![[Screenshot from 2024-04-19 01-19-46.png]]

![[Pasted image 20240419012002.png]]

C (max is 52%)
![[Screenshot from 2024-04-19 01-20-22.png]]

Cache --> 

Decreasing Cache can be done with:
* evicting values till I reach the original Cache size. 
* Replacing the fake values in a batch ($F_D$ ) with these. 
* For example, C=10 then I add 2 --> N=N+2 and C=12
* I go back to N ==> C=10 --> 2 After eviction
* B is now R+($F_D$-2) so that I can also write back whatever is being evicted out.


Think about: 
* Are there any edge cases for this? Does D decrease? Are we compromising security in any case. 
* What happens to the dummy values we were about to write? Does their access frequency change at all?

## April 26 

Some edge cases:
1) Small values of $F_d$ :
	* Case where cache sizes grows above $F_d$, its possible we stop adding $F_d$ for a few batches. Can possibly leak information (alpha for dummy value exceeds bound). 
	* Solution: Never exceed some % of $F_d$ to replace with cache evictions. 

2) Right now algorithm sets the timestamp of all dummy objects to the same value after $D/F_d$ accesses for randomisation. Maybe we can have a workload where this randomisation happens after a very long term (long enough to reveal a pattern?).
	1) I don't think this would be an issue before of re-encryption? 
3) 



Perf report of proxy_server with core = 1


![[Screenshot from 2024-04-23 19-03-57.png]]

![[Screenshot from 2024-04-23 21-45-40.png]]



No encryption/decryption bottleneck. Main bottleneck is the BST. Can't make a lock free BST because you need to sort on every insert/delete. 


![[flamegraph.svg]]


What I did: 
* Tried starting over implementation:
	* Realised this will clearly take a lot more time since there are a lot of things going on, and some new programmatic concepts I would need to cover (like Promises and Futures in c++ --> I know what they are but not within c++). 
	* Started looking into the proxy and capturing some performance metrics. One thing that stood out was the % of cpu taken up by the bst function `setFrequency`. 
	* Looked into its implementation. The current Implementation:
		* Uses an unordered hashmap to store and get frequencies of keys. This is basically O(1)
		* Uses a <\set> (Balanced Binary Tree) with a compactor function to get key with min Frequency O(log n).
		* Wrote and Ran a simple benchmark for setFrequency:
			* BST size: 1.2 Million
			* Pretty good performance.
				* For single threaded ~ 3 Million Requests/Second.
				* For 2 threads ~ 2 Million/s
				* For 3 threads ~ 1.5 Million/s 
				* For 4 threads ~ 1.7 Million/s
		* I wanted to see if I could improve performance of just this component:
			* Tried using a linked-list instead of Set.
			* Tried Using only the unordered map and then sorting.
			* Replacing Set with Ordered Map
			* Lazy Sorting (Only sort when I need to).
		* None of these worked and all resulted in a throughput <1000. 


Overall I think, after experimenting for ~1.5 week that It'll be better (and faster) to use this implementation as is and build on top since it is already pretty well written. I can polish out any bugs as I go along but I think this should be self-contained. 

	As the project is right now, It's pretty easy to start and deploy (Although I need `to make a few changes to the CMakeList.txt and fix dependency issues). So we can target the badges easily. 




## May 10

Load Balancer: 
* Basic implementation done.
* Works on remote machines (Load balancer connects to different executors and initializes them).
* Experiments:
	* Experiment one: Single machine, Default Benchmark (One provided in git repo). Redis on localhost. Number of Clients=1
		* Throughput: 10239 op/s
	* Experiment Two: Same as Experiment one, Number of Clients = 2
		* Throughput: 10286 op/s
	* Experiment Three: 
		* Machine 1: Load balancer --> Initializes both executors and launches a thread to benchmark each executor.
		* Machine 2: Executor 1 --> Connects to local redis server
		* Machine 3: Executor 2 --> Connects to local redis server
		* Throughput: 29704 op/s  --> ~x2.8 speedup
* All experiments for Async client. 



### May 17th

Shortstack:

![[Screenshot from 2024-05-15 23-20-13.png]]
![[Screenshot from 2024-05-15 23-20-46.png]]

Each layer forms a connection in the forward and reverse direction, (One socket for each # of workers in a layer). Propagate acknowledgements backwards. Clears state on when a particular request has been served. They can use simple sockets since the # of workers won't be a real bottleneck? They 3 workers in each layer as an example.



#### Problem:
1) Once the query resolver starts an operation (Select, Join, Update) and does an index call, it passes the key/value pair(s) to the load-balancer and waits for the responses back. 
2) Once we get the responses back from the load-balancer, how do we know which threads do these keys belong to? How do we redirect the keys to that thread. 

### Possible Solutions: 

##### Context
Parser converts the SQL statement into an object of type:
```cpp
select age from table1 where name = haseeb
struct parsedQuery{
	string client_id="123";
	string type = "select";
	string tableName = "table1";
	vector colToGet = ["age"]; //Can we multiple columns
	string searchCol = "Name"; //Can filter on a single column.
	string searchVal = "Haseeb";
}
```

###### A:
Construct a request object for the load-balancer:
```cpp
struct loadBalanceRequest{
	int request_id;
	int object_id; 
	vector keys = []; //Fill it with keys
	vector values = []; //Respond back by filling in values.
	num_objects = 2; // Say we split the request for 100 keys into two objects of 50 each. 
	bool finished=false;
}
```

When a request comes in, we launch a new thread that gets assigned a queue within a concurrency friendly map. The thread can directly call the load-balancers get interface but has to wait on to get the indexed key back. It then generates the keys for the request and waits once more. 

```cpp

void SelectFunction(request){
	index_key = request.getIndexKey() //Generate Index key
	fetchKey(index_key)// Call Loadbalancer to get the key

	while(queue_map[ThreadId].size()==0{}; //Busy wait. 
	pk_range = queue_map[new_request.id].pop();
	keyRange,num_objects = generateKeyRange(pk_range) //generate the keys for the object we want.
	
	fetchKey(keyRange)//Push to load balancer again. This would be the loadBalanceRequest object. 
	
	while(queue_map[ThreadId].size()<num_objects){} //Wait for all objects to come into queue. 

	vector all_pairs;
	for i in num_objects:
		obj = queue_map[new_request.id].pop();
		kvPair = makePairFromObject(obj); //Makes the loadBalanceRequest and make a pair of the k/v lists.
		all_pairs.push(kvPair);

	respondToClient(all_pairs,reques.client_id);
	erase(queue_map[ThreadId]); //Cleanup queue memory. 
}


void runQueryResolver(){
	new_request = recieveQueue.pop() //Get a new request (parsedQuery) object
	queue_map[new_request.id] = new Queue. 
	
	launchNewThread(new_request,SelectFunction) // Assuming we only have select requests, launch new thread.
} 

void loadBalancReadThread(){
	while(true){
		request = loadbalanceRecieveQueue.pop();
		queue_map[request.id].push(request);
	}
}

```

In this case, the thread keeps state of the request and issues/receives `loadBalanceRequest` objects. The load balancer will be responsible for maintaining the object.  (Can this get tricky for repeated keys, or keys that overlap between objects?)

1) Get Request from Parser 
2) Launch Thread that is responsible for handling query end-to-end. 
	1) Issues Index Request (if Any)
	2) Waits for it to respond back.
	3) Issues request for Keys it wants.
	4) Waits till it gets all objects. 
	5) Responds back to client. 
3) One socket/connection between query resolver/load balancer. 


###### B:
Construct a request object for the load-balancer, but also include the `parsedQuery` object.
```cpp
struct loadBalanceRequest{
	int request_id;
	int object_id; 
	vector keys = []; //Fill it with keys
	vector values = []; //Respond back by filling in values.
	num_objects = 2; // Say we split the request for 100 keys into two objects of 50 each. 
	bool finished=false;
	parsedQuery obj; //Contains the information for the query. Only add it to the first request object. 
}
```

When a request comes, we launch a new thread:
1) If the request will use an index, call the indexer thread. 
	1) The indexer thread only issues request for the index and terminates. 
2) The `loadBalancReadThread` :
	1) Before inserting values into the object, checks if the key already exists (if a thread is currently waiting for more objects).
	2) If not, it starts the select function (or whatever function is needed for the type). Makes a queue for it and passes the along. 
	3) The selection function gets all the information it needs and only does the `selection`and not indexing. 



| A (pro)                                                                                                                      | B(pro)                                                  |
| ---------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| Simpler. Isolated logic. <br>Thread handles query end-to-end.                                                                | More scalable and efficient. Less threads busy waiting. |
| Less state movement. Query state remains<br>at the query resolver level. Doesn't move <br>around (less data to move around?) | Possibly less contention between threads?               |
| No hassle in dealing with out-of-order objects                                                                               |                                                         |


| A (Cons)                                                                    | B(Cons)                                                  |
| --------------------------------------------------------------------------- | -------------------------------------------------------- |
| Not Scalable, We can launch a lot of threads.<br>High chance of contention? | More complicated to implement and ensure synchronization |
| Lots of threads waiting.                                                    | Overhead of thread launching/closing.                    |

##### ORAM Benchmark

Objective: 
Substitute the underlying Oblivious Key-Value store and benchmark the system. Try to answer the question:
If I used another oblivious Key-Value store (Like ORAM), how much of a performance improvement/degrade would I experience?

Assuming the load balancer calls the `sendBatchtoKV` for each executor: 

```cpp

void sendBatchToKV(std::vector<std::string> &keys,std::vector<std::string> &values){
	if(values.empty()){
		//Its a Get Request
		auto response = kvStore.get_batch(keys); //Your function gets called here via RPC.
		//response is plaintext key-value pairs. 
	}else{
		// Its a Put Request.
		kvStore.put_batch(keys,values); //Your function gets called here.
		// No response. 
	}
}

```



1) Requirement:
	* Implement an ORAM system (Path ORAM or some system similar to SEAL) that exposes `get_batch` and `put_batch` interface. 
	* `get_batch:` takes as input a vector (std::vector<std::string> keys) which is a set of plain-text keys to get from the remote server. 
	* `put_batch`: takes as input two vectors (keys & values). The remote server update the corresponding keys to the new values. 
	* Handle logic for missing keys. 
	* Should be able to deploy multiple servers on separate machines (2-3 ORAM Units?)
	* Connect to redis K:V store. 
	* Use same libraries that waffle uses (For RPC calls, connecting to redis and Encryption)



* Send SEAL + Waffle report 
* Work on LB and Query Resolver (with RPCs)
* Implement A Logic (Sync). 



* Dependency Logic at resolver
* Assign timestamp to each Request. 
* Timestamp ordering of Key Values within each request. 
* The timestamp determines which request gets which version of the value. 
* Design a protocol that keeps the versioning in mind. 
	* Same Request (R(X), W(X), R(X)) --- returns --> (V1,V2,V2).



### June 7th 

Issue:
* We need a total ordering between requests so that we can order them correctly (Request 1 happens before request 2). 

Solution:
* Have a incrementing `request_id` that acts as the timestamp for the request. 

Simplifying assumption: One request is not broken up into different requests. One Request Object is created per request. 


##### Flow of a request:
1. Generate Request 
``` cpp
loadBalancerRequest req = {
	int "request_id":1,
	int "object_id":NULL //Not Using
	int "numObjects":NULL // Not Using
	vector<string> keys : ["K1","K2","K3","K1","K1"]
	vector<string> values:["","","","Test",""] // GET GET GET PUT GET 
}

//Assuming all Keys are initialized to the value "Hello" We should get a response back like:
{
	vector<string> keys : ["K1","K2","K3","K1","K1"]
	vector<string> values:["Hello","Hello","Hello","Test","Test"]
```

2.  Request Sent to the Load Balancer: 

```cpp
struct responseGroup{  
    loadBalanceResponse response;  
    int total;  //How many are expected.
    int currentSize;  //How many responses recieved.
    
    responseGroup(int tot){  
        total=tot;  
        currentSize=0;  
    }  
};

void LoadBalanceProxy::addKeys(const sequence_id& seq_id, const loadBalanceRequest& req){  
    std::vector<std::string> keys = req.keys;  
    std::vector<std::string> values = req.values;  
    responseGroup temp = responseGroup(req.keys.size());  //Initialize Size 
  
    responseMap[req.request_id]=temp;  //State at the loadbalancer for all requests being processed.
    sequence_id_cache[req.request_id]=seq_id;  //State at the loadbalancer needed to respond back to client.

	//Hashing keys to Assign them to an executor. 
    for(int i=0;i<keys.size();i++){  
        int executorNumber = hashFunc(keys[i]) % numExecutors;  
        auto pair = std::make_pair(req.request_id,std::make_pair(keys[i], values[i]));  
        //Each Pair  = (Request_ID: (K:V))
        executorQueue[executorNumber].push(pair);  
    }  
}

```

Here the Load Balancer parses out the keys and setups state for each request. 
Executors are given the Key:Value pairs along with the request_id this key:value pair belongs to. 

3. Executing a Batch (Give a batch an id and order responses accordingly. Remove Lock)
Once all executors reach R requests, we start to execute batches: 

```cpp
void LoadBalanceProxy::executeBatch(int executorNumber,std::vector<int> reqIds,std::vector<std::pair<std::string, std::string>> kvObjects){  
	//reqIds is a list of request_ids from which the requests are coming. This is needed to signal another thread. 

    std::vector<std::string> keys;  
    std::vector<std::string> values;  
  
    for(auto &p:kvObjects){  
        keys.push_back(p.first);  //Parse Out Keys 
        values.push_back(p.second);  //Parse out corresponding Values/
    }  
  
    //For each executor, we keep a single socket (which remains open). Sending concurrent requests via that causes some  
    //Issues with thrift internals (-ve frame number). Sync requests here.//This is still incorrect since for multiple pending requests, any of them can aquire a lock (and not sequentially). ToDo: Make it so its sequential (Queue of waiting threads?)
    
    executorsRunningBatch[executorNumber].lock();  
    executor_reply reply  = executors_[executorNumber].execute_batch(keys, values);  
    executorsRunningBatch[executorNumber].unlock();  
    // std::cout<<"Executor: "<<executorNumber<<" Got response of size: "<<reply.Keys.size()<<std::endl;  
  
    std::vector<std::string> replyKVPair;  
    std::set<int> uniqueReqIds;  
  
    responseMutex.lock();  
    //Aquire lock so only one thread updates the responseMap at a time
    for(int i = 0; i<reply.Keys.size();i++){  
        int curSequent = reqIds[i];  
        uniqueReqIds.insert(curSequent);  
        responseMap[curSequent].response.keys.push_back(reply.Keys[i]);  
        responseMap[curSequent].response.values.push_back(reply.Vals[i]);  
        responseMap[curSequent].currentSize++;  
    }  
    responseMutex.unlock();  

    for (const int& element : uniqueReqIds) {  
        responseDone.push(element); //Signal that I (the thread) has dealt with these responses.   
    }
```

Note: 
We can have a case where: 
1. Requests in the same execution (distance between requests < R):
		Get K1 (Request_Id: 2) 
		Put K1 (Request_Id: 4)
		
	Both these requests get bundled to the same executor.  This is fine since the request for `Get K1` comes before the request for `Put K1` (We use a FIFO queue). The requests in the same order are sent to the executor (which is then responsible for ensuring Linearizability). 

2. Request in different executions (distance between requests > R):
	Even if different executions handle the same request, the linear order of the request doesn't change. For changes to the same Key, the request that came up first would finish first, then the next request keys would be looked at. 

The lock around the fetching of values `executorsRunningBatch[executorNumber].lock()` also ensures that we don't run two batches concurrently. The batches themselves also are ordered (the order in which the threads where started).
For example all executors queues have 10 values. If R=5 then its possible we launch two executeBatch threads per executor. This can cause a put request in the second execution to happen before a get in the first execution. The lock ensures that the first execution takes place first, and then only can the second execution take place. 


4. Sending back a response:
```cpp
void LoadBalanceProxy::checkResponsesFinished() {   
    while (!done) {  
        if (responseDone.size() == 0) {  //responseDone is a queue
            continue;  
        }  
        int seq_id = responseDone.pop();  //Pop Element (This queue will populate once an execution of a batch has finished)
        responseMutex.lock();  
        //Check if the responses have finished. 
        if (responseMap[seq_id].total == (responseMap[seq_id].currentSize)) {  
            id_to_client_->async_respond_client(sequence_id_cache[seq_id], ADD_KEYS, responseMap[seq_id].response);  //Send back the response
            responseMap.erase(seq_id);  //Clear State from the Loadbalancer for this request.
            sequence_id_cache.erase(seq_id);  //Clear State
        }  
        responseMutex.unlock();  
    }  
}
```

For example if `request_id 1` was split between Executor 1 and Executor 2. Executor 1 will push `1` into the `responseDone` queue. This loop will check and see that the responses are not yet complete. Once Executor 2 finishes, it will also push `1` into the queue. This time the loop sees that all keys within the request have been completed and sends back the response. 


Overall if we have: 
1. Two Selects running concurrently --> No conflicts.
2. Update And Select (Update then select) --> Update runs before select. (Select sees the changes made by Update)
3. Select and then Update (Select then Update) --> Select does not see the update. 

Any more edge cases we can think of?


Next steps:
* Replace naive locking with ordered thread access. 
* Add timeout logic for executor queues. 
* Implement index and select on query Resolver. Test for correctness. 


In the executor, ensure ordering by giving each batch a number and processing them in that order. Do this with the Plaintext keyvalue store and waffle

Discussion after she comes back.
* Discussion on inserts/deletes and updates on changing Outsourced N. (The discussion on dynamically changing alpha.)



## June 24

Issue: Thrift isn't thread safe. We cannot have a single service/process send multiple concurrent requests (over the same socket) to another service (RPC call). 

Solution: 

[gRPC](https://grpc.io/docs/what-is-grpc/core-concepts/)
* more recent rpc framework. CockroachDB also uses it. 
* Support for C++ as well as go. 
* Supports bi-directional streaming as well. 
* Sync and Async Calls 
* **Great Documentation**

Unary RPC: 
* Client sends a single request and gets back a single response. 
Server Streaming RPC:
* Client sends a single request and server returns a stream of messages in response. (Example: Stock Prices/Ticker)
Client Streaming RPC:
* Client sends a stream of messages to the server instead of a single message. It commits once messages finish. Server responds with a single message. (Example: Uploading a file)
Bidirectional RPC: 
* Two independent streams, Client streams messages to server and server streams messages to client. (Example: Multiplayer Gaming)

Within Go (sync): 

Each RPC handler attached to a registered server will be invoked in its own goroutine. For example, [SayHello](https://github.com/grpc/grpc-go/blob/master/examples/helloworld/greeter_server/main.go#L41) will be invoked in its own goroutine. The same is true for service handlers for streaming RPCs, as seen in the route guide example [here](https://github.com/grpc/grpc-go/blob/master/examples/route_guide/server/server.go#L126). Similar to clients, multiple services can be registered to the same server.


Within C++ (Sync):
[ref](https://groups.google.com/g/grpc-io/c/Cul6fd7cOB0)
Uses a thread pool to manage multiplexing requests. 


Benchmark:
[ref](https://github.com/chrislee87/rpc_benchmark/tree/master)
Long Connection --> One connection for a client to do all RPCs
Short Connection --> One Connection for each RPC call. 
![[Pasted image 20240624105115.png]]

$A$ = 1
$B$ = 1
$C$ = 1 

$B_1$ = [R(A),R(B),W(A),R(C)] --> Result [1,1,3,1]
$A$ = 3
$B$ = 1
$C$ = 1 

$B_2$ = [R(C),R(B),R(A)] --> Result [1,1,3]
$B_3$ = [R(D),R(F),R(X)] 


$B_1$ ,$B_2$ and $B_3$ issued in parallel. B2 Depends on B1 so Serialize as: $B_1$ ---> $B_2$. But $B_1$ and $B_3$ can run in parallel. 

Possible Solutions: 

At Query Resolver:
Stick with this solution. 
* Keeps global state on records being processed by threads (And what Keys). Thread that issues B2 waits to issue requests. (We can have possible deadlocks? Thread ends up waiting for a long time to issue the request) --> Leads to deadlock. 
* Make a precedence graph (Shared amongst threads?) Add and remove nodes as threads request and receive responses. 
	* Higher priority 


TPCC Requests --> TPC-C Look at their requests and see how that maps out with this. 
Number of Keys per query --> Construct stats on number of keys generated. 
Deadlocking mechanism. 

At Executor: 
Can't assume anything about Network. Don't use this. 
Relying on the fact that $B_1$ comes before $B_2$ to the executor. We keep state on all keys being currently written (We will only conflict if there is a W(X),R(X)). The go routine that launches for B1 inserts W(A) to this state. Go routine for B2 checks if any of its keys have membership in this state. If they do, pause until the membership expires/finishes, Otherwise continue with execution. 

Removing the assumption that $B_1$ comes before $B_2$, Executor now keeps track of request ids it has seen (they will always be incrementing). If it sees any out of order request, it launches a go routine to handle the request but waits till it can execute. 
For example:

Requests 1,2,3 arrive to the executor in the order 1,3,2.
Executor is expecting 1 (it knows the id for the next request since its current+1). It will launch a routine for req 1 and process.
Executor gets 3, it was expecting 2. It launches the routine but waits.
Executor gets 2, it was expecting this request, launches routine and processes it. It will increment current counter and this would signal the routine responsible for req 3 to process. 

Then 2 and 3 can occur concurrently if there are no conflicts. 



Come back to serialization after we are done with Selects (Ranges, Joins). (Make Note on ToDo)



1) With scaling how are you doing security.
2) Up to date load balancing techniques -->
	1) Do we call it a load balancer or something else?
	2) It is needed for security guarantees, Mainly. In high throughput scene it can be beneficial. 
3) In what cases would moving keys be beneficial. 
	1) Is it beneficial to have these keys in a separate proxy or in a single proxy. 
	2) 2 keys in one proxy, vs 1 key each proxy.
	3) Justify why we aren't moving keys, and why it works. 
4) Don't make changes to stash. 
	1) All these proxies (ORAM) have a cache.
	2) Padding to the next power of 2 in ORAM executor (SEAL).



## July 11th 


#### Big Data Benchmark:
Query 1:

```
SELECT pageURL, pageRank FROM rankings WHERE pageRank > X
```
Comments: 
* We can do this with the current logic selection function we have. 

Query 2:
````
SELECT SUBSTR(sourceIP, 1, X), SUM(adRevenue) FROM uservisits GROUP BY SUBSTR(sourceIP, 1, X)
````

Comments: 
* Substr extracts the the substring from sourceIP from position 1 to X. 
* Then groups the extracted string, and sum of adRevenue. 
* a Row might look like : 127.0 - 1000 (X = 5)
* We can do something similar by fetching the entire columns and then aggregating and grouping by?

Query 3: 
```
SELECT sourceIP, totalRevenue, avgPageRank
FROM
  (SELECT sourceIP,
          AVG(pageRank) as avgPageRank,
          SUM(adRevenue) as totalRevenue
    FROM Rankings AS R, UserVisits AS UV
    WHERE R.pageURL = UV.destURL
       AND UV.visitDate BETWEEN Date(`1980-01-01') AND Date(`X')
    GROUP BY UV.sourceIP)
  ORDER BY totalRevenue DESC LIMIT 1

An equivalent could be:

SELECT UV.sourceIP, 
       SUM(R.adRevenue) as totalRevenue, 
       AVG(R.pageRank) as avgPageRank
FROM Rankings AS R
JOIN UserVisits AS UV
  ON R.pageURL = UV.destURL
WHERE UV.visitDate BETWEEN Date('1980-01-01') AND Date('X')
GROUP BY UV.sourceIP
ORDER BY totalRevenue DESC
LIMIT 1;


```
Comments:
* Scanned through ObliDB source code, This query is broken down into functions I think, can't exactly point to where this query is happening. Inferred this from the test file that is available. 
* Few points in this query:
	* Join is within a sub query,
	* Aggregates upon a join. 
	* We can convert the date to a range query. But we need to cater to the date type (day wraps around and increments month)
	* We can do this by:
		* First using join map on pageUrl & destUrl, fetch index for visitDate (range to point queries). Take out intersection.
		* For these primary-keys, fetch source ip, pageRank and adRevenue, group by source IP and calculate aggregates
		* Order the items by totalRevenue and return the first item.


#### TPC-C 

#### New Order Transaction 

```
SELECT C_DISCOUNT, C_LAST, C_CREDIT
    FROM customers
    WHERE C_W_ID = ?
    AND C_D_ID = ?
    AND C_ID = ?

SELECT W_TAX
	FROM warehouses
    WHERE W_ID = ?

SELECT D_NEXT_O_ID, D_TAX
    FROM district
    WHERE D_W_ID = ? AND D_ID = ? FOR UPDATE
```

For Update locks the selected rows for a transaction. 

```
INSERT INTO newOrder
    (NO_O_ID, NO_D_ID, NO_W_ID)
    VALUES ( ?, ?, ?)
         

UPDATE district
    SET D_NEXT_O_ID = D_NEXT_O_ID + 1
    WHERE D_W_ID = ?
    AND D_ID = ?

INSERT INTO openOrders
    (O_ID, O_D_ID, O_W_ID, O_C_ID, O_ENTRY_D, O_OL_CNT, O_ALL_LOCAL)
    VALUES (?, ?, ?, ?, ?, ?, ?)

SELECT I_PRICE, I_NAME , I_DATA
    FROM item
    WHERE I_ID = ?

SELECT S_QUANTITY, S_DATA, S_DIST_01, S_DIST_02, S_DIST_03, S_DIST_04, S_DIST_05,
    S_DIST_06, S_DIST_07, S_DIST_08, S_DIST_09, S_DIST_10
        FROM stock
        WHERE S_I_ID = ?
        AND S_W_ID = ? FOR UPDATE

UPDATE stock
    SET S_QUANTITY = ? ,
        S_YTD = S_YTD + ?,
        S_ORDER_CNT = S_ORDER_CNT + 1,
        S_REMOTE_CNT = S_REMOTE_CNT + ?
    WHERE S_I_ID = ?
    AND S_W_ID = ?

INSERT INTO orders
	    (OL_O_ID, OL_D_ID, OL_W_ID, OL_NUMBER, OL_I_ID, OL_SUPPLY_W_ID, OL_QUANTITY, OL_AMOUNT, OL_DIST_INFO)
        VALUES (?,?,?,?,?,?,?,?,?)
```

Rest are here: 
https://github.com/cmu-db/benchbase/blob/aee8929cfe2fd34335f18c505b7303b7e7cda243/src/main/java/com/oltpbenchmark/benchmarks/tpcc/procedures/Delivery.java

Comments: 
* Most of the queries follow the same pattern (selections of a bunch of columns using filters)
* Stock count transaction has a count distinct aggregate
* Delivery has a delete query and a sum query (both using filters)
* No Joins. 


Benchbase was used by Obladi (it was previously called oltp bench, https://github.com/oltpbenchmark/oltpbench/). There is a paper on this as well which we can cite. Benchbase is the newer version. 

Other benchmarks from Benchbase:
1. Epinions:
	1. https://github.com/cmu-db/benchbase/blob/main/src/main/java/com/oltpbenchmark/benchmarks/epinions/procedures/GetAverageRatingByTrustedUser.java
2. Hyapt:
	1. https://github.com/cmu-db/benchbase/blob/main/src/main/java/com/oltpbenchmark/benchmarks/hyadapt/procedures/MaxRecord1.java
3. OT-Metrics:
	1. https://github.com/cmu-db/benchbase/blob/main/src/main/java/com/oltpbenchmark/benchmarks/otmetrics/procedures/GetSessionRange.java
	2. This has an IN clause, which we can also convert to a set of point OR filters.





Convert tracefile to epinions database.
Use that dataset and queries as a template and create selection function




`SELECT avg(rating) FROM review r, trust t WHERE r.u_id=t.target_u_id AND r.i_id=? AND t.source_u_id=?"` --> Aggregate over join
`SELECT avg(rating) FROM review r WHERE r.i_id=?` --> Agg over point query

`SELECT * FROM review r, item i WHERE i.i_id = r.i_id and r.i_id=?  ORDER BY rating DESC, r.creation_date DESC LIMIT 10;` --> Join query with an Order BY.

`SELECT * FROM review r, useracct u WHERE u.u_id = r.u_id AND r.u_id=? ORDER BY rating DESC, r.creation_date DESC LIMIT 10` --> Join query with an Order BY
Perf difference between the two joins (Maybe different table sizes?)
As is



 -->Simple Selection with Order by
Change this to --> `SELECT * from review r where creation_date between <Day 1> and <Day 2> ` --> Range query on creation_date. (Add index on creation_date)


Done: 

`SELECT * from review r where creation_date between <Day 1> and <Day 2> ` --> Trade off of range to point. 
`SELECT avg(rating) FROM review r WHERE r.i_id=?`
`SELECT * FROM review r WHERE r.i_id=? ORDER BY creation_date DESC`

Do Next:



//Finish up these joins with Join map, and then work on join with one item with index on the join column.




Possible Benchmark:
* Memory Benchmark of Postgres and our resolver program. Hopefully Memory footprint of resolver << Postgres (In Memory storage)
* 