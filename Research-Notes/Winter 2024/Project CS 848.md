

We want to run evaluations in order to answer two questions: 

1) At what stage does it become more expensive to run the two round protocol. For what table sizes would it make more sense to just retrieve the entire table rather than doing a two round protocol? --> This helps us with query planning. (Depending on selectivity of the query, is it easier to just pull the entire table from server?). 
2) Do we gain a performance improvement by scaling the executor nodes? 


What to do:
##### Question 1: 
1) Use tpcc-generator to get csv files for tpcc https://github.com/alexandervanrenen/tpcc-generator
2) Parse them into key:value pairs. 
3) Write an indexing interface. 
	1) This will interact with the waffle code-base as is. We can refer to the proxy_benchmark and see how it initialises the DB and does gets/puts. 
	2) We can generate Key:Value pairs and also generate index for the an attribute we want to test for (lets say age). 
	3) Start proxy with this dataset. Generate an Input we give to the indexing interface.
	4) Benchmark how long it takes to: 
		1) Fetch the index.
		2) Fetch the values. 
	5) Run another benchmark that just fetches the entire table. 
	6) Measure latency for increasing table sizes. 

Evaluation graph: 

* Latency on Y-axis
* Table Size on X-Axis
* Two lines, one is for with indexing and one is without indexing(full table scan + filtering). 




##### Question 2:

1) Write a simple load balancer. 
	1) This will have a queue for each executor.  --> Queue size is R. 
2) Assign each executor 1M keys. 
3) Run a simple K/V get from loadbalancer. 
4) See throughput/latency 
5) Do the same with 3M on single executor. 


Case 1:
Fetch entire table --> Filter for Value
Fetch Just search Column --> Filter --> Ask for second column 
Fetch Index --> Parse index --> Fetch just the values. 



