
### Goal: 

Design a new relational (SQL) database system that protects against access pattern and volume-based attacks in the Client-Server data storage model. The client should be able to do updates to the database (we don't want a static database) and We want to allow the user of this system to modify and change security parameters according to their requirement.  The database should be performant and have reasonable throughput with support for concurrent users (connected to a proxy). To this extent, we use Waffle as a foundation for hiding access patterns and volume when fetching or writing data to an external server. The system should support a variety of SQL queries such as Select, Insert, Update, Join, and Where. We also need to support range queries over numeric data. Our main contribution is to produce a relational database system that: 
1. Protects against access and volume pattern attacks. 
2. Allows system admin to control the level of security. 
3. Deployable on any system, we don't want to use (TEE). 
4. Ideally more performant than previous implementations (SEAL and ObliDB). 

### Background: 

#### What Waffle Does: 

Waffle introduced 𝛼, 𝛽 uniformity. In summary (within the context of a key/value database), this uniformity states that:

1. 𝛼 is an upper bound that ensures any key that is written to the server, will be read after at most 𝛼 server accesses. As an example, if 𝛼 = 5, this means that every key present on the server must be accessed within 5 server accesses. 
2. 𝛽 is the lower bound that ensures any key that has been read from the server, will be written back to the server after 𝛽 accesses. As an example if 𝛽=10. A key that has been read from the server will have to wait a minimum of 10 server accesses to be eligible for writing back to the server. Having a large B is beneficial, since we can delay writing back to the server (and hence reading it back). 

Waffle operates in batches, so multiple (read/write) requests are batched together and sent to the server. 

This combination of batching, along with the read-once protocol, provides an oblivious storage system because: 
1. You are batching Real Requests, Fake Requests on dummy objects, and Fake Requests on Real Objects, this hides from the server what you are requesting. 
2. The key/value pair is deleted after being read once. The server would not see the same key/value pair again. It's harder for the server to maintain associations/correlations as you are accessing a key only once. 
3. The system also hides volume patterns as the result set is masked by the batch size. For example, if we have a batch size of 100, the server doesn't know if the result set is 100, 80, 20, or 1. The filtering is done on the trusted proxy. The server only gets 100 requests and replies with 100 values. 

#### Waffle - Existing Data Structures:

Waffle employees the use of BST (Binary Search Tree) to maintain access timestamps, one exists for Real Keys (Real Objects) and one exists for Dummy keys (Dummy Objects). 

### System Design:

##### These choices are open to changes, both small and drastic. 

#### Intro: 

We are building a relational database, so we would like to support basic SQL operations. These include Select, Insert, Update, Join, and Where. We would also like to support range queries over numeric data. 

#### Using Redis as a backend storage

Although Waffle is a Key/Value store that employs Redis as the backend storage service, using a similar key/value store to implement a relational database might not be straightforward. We would need to: 
1. Translate a SQL query to a corresponding Key set. We would need to write a custom SQL parser. 

#### Using MySQL as backend storage

As an alternative, we can use MySQL as the backend storage service and delegate some of the storage optimizations to the server itself. A good example of this is `select * from table1;`. In the case of a K/V backend, we would need to implement a function that translates this SQL statement to a set of Keys. If, instead, we use a MySQL backend service, we can just forward this query without any parsing. (More on this later).

#### Elaborate on MySQL as data storage Layer

In this case, we would have a MySQL database, where each table would be a Waffle Instance (Having independent Alpha and Beta Values). This abstraction (of 1 Waffle = 1 Table) would be at the proxy, for the MySQL datastore, it is one database, with multiple tables (across the same or different schema).  At the proxy, each Table would have its own Waffle Access Timestamp BST on Real Objects, Dummy Objects and any indexes we create.

Index I was drawn out separately, this can be a different server running a Key/Value store or just another table in the same MySQL database, we can experiment and see what is better. 

At a high level, we want to take a read query and parse it down to:
`select <col> from <table> where id in <array>` and forward this to the MySQL Data storage layer. 

#### Index: 
Further down in the text, I mention "Index on Col1" or "Use Index", let me first explain what this index really is. 

Lets say we have a table with the column "Age". The user specifies that an index be created on this column (This will need to be provided by the system admin). 

When this table is added to the database, the proxy will create an index for itself. This will just be a map, which maps from Key Space (Unique values of Age) K --> V (Document Ids for that have this Unique Value of Age). 

Similarly if the column is "Role". A similar map will be created. These indexes can be stored on the proxy along with a timestamp for each value, helping us know which ids to issue requests for (Fake requests on real objects). 

For Range Queries, we will use something similar to the Logarithmic BRC/URC technique covered by Ioannis's paper. This can be implemented as a simple segment tree or a range tree. This tree will also hold nullable timestamps to help us with retrieval and also notify the proxy of the presence of document_Ids in a range (The timestamp of the node for a non-existent range will be null).

[Practical Private Range Search Revisited](https://dl.acm.org/doi/10.1145/2882903.2882911)


#### Select

* Select * from Table1: 
	* This would just be the same sql query passed to MySQL. (No BST or Indexing)
	* Modification to Waffle: We don’t delete or write back the entire table. Access pattern for all keys remains the same for that table.
	* Index Used: **None**.

* Select col from table1: 
	* Same Query passed to MySQL.  (No BST or indexing) 
	* Same modification as above. 
	* Index Used: **None**

* Select col2 from table1 where col1=10 
	* First query index  for K=(Col1|10), parse out doc_ids. 
	* Send select col2 from table1 where id in <doc_id_array> ← This would have D+f_r too. 
	* Index Used: Index on Col1.

* Same idea for Range as well. 
	* Index Used: Range Index on the Column we are querying for. 

#### Insert 

* Insert into table (Col1,Col2,Col3) Values (x,y,z);
	* Generate Key and add to Waffle BST at proxy for table1. 
	* Add to write batch (A transaction of insert requests).
	* Assuming Col1 has an index, we will first fetch (col1|x) from index and write back updated values.

Caveat: Simple Waffle (Key/Value Store) takes as Input the # of dummy objects. We would need to scale dummy objects. Can we create # of dummy objects as a function of real objects and some other parameters?
Interesting Experiment: How does the # of dummy objects affect security and performance? Can we find some optimal value as a function of the current true table size. (True Table --> Size of only Real Objects). 

#### Update 

* Update table1 Set Col1=20 where col2=y 
	* In case we don't have an index on col2, we will need to stream entire column, modify and then write back to the table. 
	* In case we do have an index, first we index doc_ids fro index table and then fetch those doc_ids. 

Joins: 
* Select * from t1 join t2 on t1.col2=t2.col2 
	* Assuming we don't have any index: 
		* Stream both tables (Select * ) and join locally. 
	* In case of pre-compute joins, first fetch the document_ids relevant for this join and then fetch those documents. 
* Select * from t1 join t2 on t1.col2=t2.col2 where col3='X'; 
	* Assuming we don't have any index:
		* Stream both tables (Select * ) and join locally. 
	* In case of a pre-compute join, first fetch document_ids for the relevant join and fetch those documents. Filter col3='X' locally. 
	* In case there is an index on Col3, we could possibly find the intersection of document_ids from the pre-compute and col3='X' document_ids. Experiments will show what is faster. 

#### What do we leak? 

1. Relative Table sizes (Table 1 is bigger than Table 2)
2. Column Names (we are not encrypting them).
3. Correlation between index and Table Access (If index is a table in the same database).
4. Database Schema. 
