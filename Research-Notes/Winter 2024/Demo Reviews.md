### Looking Deeply into the Magic Mirror

* Demo presents tools that allow database administrators to automatically setup, and evaluate a range of index selection techniques. It should provide automatic setup of the database, workload and cost evaluation. 
* Evaluation breaks down which indexes are used for which queries and associated cost of processing. 
* One can adapt the resulting index selection and observe impact. 
* Hard to determine workload cost under various index subsets (Combination of indexes). We usually estimate workload cost under a given (static) index set. 
* Dynamic workloads/changing data and index selection robustness (workload cost under given index configuration don't degrade much if workload changes) are also challenging 
* Proposal:
	* Evaluate various index selection approaches for different workloads. 
	* Automatic setup of DB, workload and cost evaluation. 
	* 8 selection algorithms. TPC-H and TPC-DS workloads and also Join Order Benchmark. 
	* Web app for analysis. Summarizes the benchmark results, enables analyzing selected indexes and index benefit per query. Allow users to adapt the index selection and investigate impact on processing costs. 
	* Open source platform, can be extended to additional workloads, databases and selection approaches. 
* Demonstration: 
	* Why doesn't every index in summary have same starting point? (0 index storage?). That would be the baseline of no indexes at all? 
	* Name of graph: Without indexes? 
	* Estimate cost per query, baseline color too similar to background. 
	* Re-evaluation runs again over the database or is it already done?
	* Demo experience could be expanded on a little more. 
		* Can run through setup by initializing a very small database (scale factor of 0.1 or 0.5?). This can allow the user to get an end-to-end experience of launching the benchmark.
		* Provide pre-computed files after this. 
		* The experience still seems a bit abstract. Defining an overall structure with a starting point and an end goal might be helpful as well? (Present a new use-case for the company and find an appropriate index for queries on this use-case).
		* Introduce 1-2 queries that you can add yourself on the dataset. Also allows the user to understand how to modify the system for their own needs. 


### QueryShield

* Data owner's data remains private.
* Works on relational + time-series dataset.
* Abstracts away MPC from users
* 3 Different demonstration scenarios. 
* Good motivation of problem in intro
* Provides functionality as a service in the cloud. 
	* Easy to use for data analysts and data owners
* Automates MPC setup at the data owners system. 
* Open source
* Build on top of Secrecy and TVA. 
* Assumes that analyst has an idea about threat models?
* Section 2 wrong ref for TVA
* 