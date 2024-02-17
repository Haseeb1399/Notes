
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


$d/f_d$ = $N-C/B+F_R$ --> Basically see how to calculate d with the above technique, 

Aggregate batch from each executor into one large so server sees one batch together. 

Partition Alpha (Server sees Alpha, each proxy issues Alpha/3 in case we have 3 executors. )

Epoch Based

Issue: One proxy runs faster than the other and then we cannot give a bound to the **server** about alpha. Even though executor pulls k1 after every 30 (Alpha), the server sees that its being accessed after 300 server accessess. 

Collect B/3 batches and then send B (Single. )

Distributed Proxy system 

Index is separate and then think about load balancing the system. 

Underlying problem: When we partition state across proxies, we don't want one to make 60 accesses (while still maintaing alpha/3) while other proxy is still lagging behind. 