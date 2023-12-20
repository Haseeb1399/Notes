

## Key-Value (Redis):

How it looks: 
#### Key|Value (Waffle A)
--------------
Col1|Value | (id1,id2,id3)
<hr>

#### Key|Value (Waffle B) - for each table.
--------------
abc | (Col1,Col2,Col3)
<hr>

Queries: 
*Create Table:*
1. Initialize a new Keyspace in Redis
2. Create a new BST for table and any indexes specified. 


*Select:*
1. Select * from table1. 
	1. This would be translated to get_batch of all keys belonging to table1 BST. 
	
2. Select col1 from table1
	1. This would be translated to get_batch of all keys belonging to table1 BST.
	
3. Select col2 from table1 where col1 = 10
	1. First send Get request to Waffle A for Col1|10, parse out the keys, and send get_batch to Waffle B

*Insert:*
1. Insert into table1 (col1,col2,col3) values (x,y,z)
	1. Generate Key and Add to proxy bst.
	2. Add to write batch key|(col1,col2,col3)
	3. Assuming Col1 has index, we will first fetch (col1|x) and then update and write back. 

*Update:*
1. Update table1 Set col1 =20 where col2=y
3. In case we don't have an index on col2, then we will need to stream the entire table, and write back the new table with updated value points.
4. In case we do have an index on col2, first fetch data from Waffle A and then Waffle B. Write back to Waffle B the updated values. 

*Join:*
1. Select * from t1 join t2 on t1.col2=t2.col2
	1. Assuming no index on col2, we will stream T1 and T2 both (issue a select * for each table) and join locally. 
	2. If we have an index on col2, then we first request values from waffle2 (using the bst we have saved for col2) and then stream the required k/v pairs and join locally. 
	3. In case of pre-computed joins, we first fetch index from Waffle A (Key would be T1.Col1|T2.Col2) and then issue a batch get for all keys across both tables to Waffle B. 

Translation: This is done by a parsing layer that we will have to write for each type of query we support for a benchmark. 

## SQL: 

Table1: 

#### |ID|Col1|Col2|Col3|
---------------------------
abc |x|y|z


*Create Table:*
1. Forward Create Table command to database (Plaintext columns?)
	1. We can keep columns as encrypted values too. 
2. Create a BST for any specified Index. 

*Select:*
1. Select * from table1. 
	1. This would go to the proxy and the proxy will forward it as is.  
	2. Note: We are not following the alpha-beta protocol here, since it touches the entire table, the access pattern for each row remains same.
2. Select col1 from table1
	1. This would also go to the the underlying sql database as is.
	2. Note: We are not following the alpha-beta protocol here, since it touches the entire table, the access pattern for each row remains same.
3. Select col2 from table1 where col1 = 10
	1. Generate a request to Waffle A for Col1|10 (We can keep this as a key/v store). Parse the keys from response. 
	2. Response_keys + Dummy_Keys + (F_r Keys) using BST = key_array
	3. Send to server: Select Col2 from table1 where col1 in key_array
	4. Follow Waffle protocol (Save in Cache, delete from server, write back)

*Insert:*
1. Insert into table1 (col1,col2,col3) values (x,y,z)
	1. Generate Key and Add to proxy bst.
	2. Encrypt x,y,z, send to server: Insert into table1 (col1,col2,col3) values (e(x),e(y),e(z)).
	3. Assuming Col1 has index, we will first fetch (col1|x) and then update and write back with new id. Delete the old (col1|x) following delete logic (replacing with dummy).

*Update:*
1. Update table1 Set col1 =20 where col2=y
	1. Fetch keys for (col2|y) from waffle A --> key_array (Indexing part)
	2. Add dummy keys to key_array (not F_r). 
	3. Issue table1 Set Col1=Enc(20) where col2 in key_array
2. Update table1 Set col1 = 20
	1. Issue table1 set Col1=Enc(20)

*Join:*
1. Select * from t1 join t2 on t1.col2=t2.col2
	1. Assuming no index: 
	Issue: 
		 Select * from table1
		 Select * from table2 
	Then join locally on proxy. 

	2. Assuming Index (Pre-computed):
		1. Fetch the index from waffle A (T1.Col1|T2.Col2) 
		2. Issue two select statements:
		Select * from table1 where id in (key_array_table1)
		Select * from table2 where id in (key_array_table2)

		Obviously dummy_keys and F_R are added to key_array_table1 




Benchmarking:


Oblidb use data from big data benchmarking tool from Berkley. Looking at the ObliDB code, they have tests for Joins,Inserts etc, I couldn't find any SQL that was being given to the tool and parsed and executed.

SEAL: Uses TPC-H, but do not provide evaluation for group-by queries. So I think they modify the TPC-H queries a bit? Because they look like this. 
![[Pasted image 20231218201510.png]]

* What do we do for functions like Interval etc, do we pass the SQL through a parser that computes all of these? 

* Do we create our tool that works for a certain benchmark? (TPC-H for example) and skip queries that we don't support? (For Example LIKE if we don't add it?)

* Do we do something similar to ObliDB and create a subset out of a benchmark and test against that. 

* What about nested queries? A lot of TPC-H is nested queries. 