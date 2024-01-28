
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

