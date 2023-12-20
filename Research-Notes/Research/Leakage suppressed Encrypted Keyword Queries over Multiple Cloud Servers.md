


S = (S1,...Ss) is a group of cloud servers

K = (k1..Km) is a set of all keywords containing in the record collection R to be searched. |X| is the number of keywords in a batch of record updates and |Y| is the number of keywords in a batch of searching. 

DB(K_v) is the search result set of keyword K_v including all identifiers of records containing K_v. 

C : Total number of update batches where each batch process l records and each record contains a different combination of keywords. 

B_v : Response vector with L elements describing record distribution of keyword k_v in the each batch of update. 

B is the binary matrix indicating the keywords contained in the entire records where B_s indicates the search result (i.e access pattern) assigned to each cloud Si. 

B[:,v]: is a bitmap array describing the query result of keyword K_v, deriving from the vth column of matrix B



Its hard to hide volumn:
Currently for a given keyword, place record identifiers of a given keyword into multiple blocks with a total fixed size. The client uses the maximum volume N to retrieve the results of all Keywords. Volume differences b/w keywords can bring extra communication cost and require client to do heavy post-processing. 

Also current schemes designed for static databases. You know maximum volume in advance. Dynamic setting you don't know that information. 


Access pattern still leaked, as locations of retrieved results are revealed. 

Do updates and Keyword searches in a Batched manner. Trade-off query response time for the leakage suppression. 



l = 12 

e = 12/4 = 3 --> 3 records in each index.

for every record 
1. [0,0,0,0,0,0,0,0,0,0,0,0] + Add 
2. E = L/Batch size --> # of records compressed in each index. 
3. [0,0,0,0,0,0,0,0,0,0,0,0] 

Reduces Length of response from 8 to 4 