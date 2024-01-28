

1) CockroachDB: https://www.cockroachlabs.com/blog/cockroachdb-on-rocksd/
2) TiDB: https://docs-archive.pingcap.com/tidb/v2.1/architecture
3) YugaByteDB: https://docs.yugabyte.com/preview/architecture/layered-architecture/
4) Spanner: https://storage.googleapis.com/gweb-research2023-media/pubtools/pdf/acac3b090a577348a7106d09c051c493298ccb1d.pdf


Related posts:
1) Rust Implementation of RelationalDB on top of RocksDB(K/V store): https://blog.petitviolet.net/post/2021-05-25/building-database-on-top-of-rocksdb-in-rust
2) https://sqlglot.com/sqlglot.html#formatting-and-transpiling
3) https://tobikodata.com/semantic-understanding-of-sql.html
4) https://shardingsphere.apache.org/document/4.1.0/en/features/sharding/principle/parse/
5) https://sidhm12.medium.com/distributed-sql-with-cockroach-db-architectural-overview-a2b7c56f8d3d

<hr>

### The parsing process. 

1) Take a SQL statement, you will first parse it: 
	1) This involves breaking the SQL token into tokens. 
	2) Checking if the SQL syntax is correct. 
	3) Checking if the columns and tables exist 
2) The SQL statement will then be formed into an AST (Abstract Syntax Tree)
	1) Here the SQL is represented as a Tree.
	2) The SQL Query Optimizer takes this tree as an input and generates a "close to" optimum execution plan. 
3) The Execution engine then runs the optimized query. 



### CockroachDB: What is does:


1) **Parsing:** Will first take an SQL statement and parse it against a YACC file (its a file that describes supported syntax etc) and generate an AST
2) **Logical Planning**: 
	1) Check if its valid SQL (Correct grammer and syntax)
	2) Resolve Names of Tables or variables 
	3) Constant folding (Convert $0.6+0.4$ to $1$)
	4) It will try to simplify the AST (Convert attr BETWEEN b AND C => a>=b and a<=c)
	5) Optimize the query using a search algorithm (cost-based optimizer)
		1) Estimate how much time each node is going to take in the query plan
		2) Model how data flows through the plan
		3) Most important factor: Cardinality of rows. Fewer rows = faster query. 
		4) Generally, this is where the Query is optimized from a "Logical POV", that its better to join A and C before A and B in (A join B join C). 
3) Physical Planning: 
	1) Helps decide which node (in the distributed architecture will participate)
	2) Not relevant to us right now. 
	3) Generally, this is where the decision for what type of join to do happens, which indexes to use, how do you access the table (Index scan, full table scan etc). 

4)  Execution : https://www.cockroachlabs.com/blog/distributed-sql-key-value-store/


<hr> 

### EncKV

