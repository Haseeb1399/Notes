
Class Link: https://cs.uwaterloo.ca/~smaiyya/cs848/


Class 1 (January 9 2024)

Intro class -- Nothing to note down. 

<hr> 

Class 2 (January 11 2024)
* Joy of Cryptography: Chapter 0 & 2 Review 

<hr>

Class 3 (January 16 2024)

#### PathORAM
1) Is there an assumption in this construction that the underlying plaintext data never changes? Would access pattern attacks / volume pattern attacks still be viable if the database was dynamic in nature? (High write workloads and low read workloads). 


* Workload Independence: 
	* Hide which data is accessed,
	* How old it is,
	* Access Pattern (Skewed vs Uniform)
	* Whether Same data being accessed. 
	* Whether access is a read or a write. 


<hr>

* Shorter summary, more on analysis and weaknesses and 
	* "This can be improved, by doing so and so"
* Justify why its a weakness.





<hr> 

Class 5 (January 23rd 2024)

Inference Attacks on Property-Preserving Encrypted Databases. 


Interesting point:
1) Frequency and Lp perform equally well for p=2,3 
	1) Does this mean that the histograms were all strictly ordered? 
2) Would it still work if full DOB is stored instead of birth year. 


<hr> 



<hr> 
## Obladi 


### What is the problem?

Supporting ACID transactions while guaranteeing data access privacy. 
Existing ORAM isn't fault-tolerant. Durability requires stash to be recoverable. 
Current techniques used in DBMS for ACID make hiding access patterns more problematic.
Improved Failure Model: Proxy and application clients may fail. 

### Why is it important?

Transactions: Can extend Obliviousness to OLTP and OLAP applications. This means that a new set of applications can use hide access patterns. 

Define ACID and that these improve the functionality of the database system itself. Durability is needed for making sure what you commit to the server persists. 

### What is it hard? Why don't previous methods work?

First work, no previous work has been done. 
Techniques used to guarantee ACID properties inherently add structure to ordering of reads and writes. This has additional information leakage. 

Concurrency Control: 
	1) 2PL: Delays write operations, by giving a txn a lock. Others have to wait for this lock to be released. Adversary knows if there is a deadlock or a highly contended item is being accessed. 
	2) MVCC: You can get cascading aborts (t1 has to abort because t2 aborted).

Failure Recovery: 
	1) Atomicity requires that uncommitted items be rolled back.
	2) Durability requires that effects of committed transactions should be preserved 
	3) Done using Undo and redo logs. Undo logs --> Reveal operations done in a txn. 

### What is the solution to the problem the authors propose?

1) Overview of RING ORam 
2) Proxy Logic: Batch Manager and Data Handler
3) Configuring Obladi
4) Parallelize the ORAM 
5) Durability, 
6) Security. 

### What interesting research questions does the paper raise?


### How does the solution perform?

Evaluations 

## Opaque 
