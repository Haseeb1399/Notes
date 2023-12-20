
#### Overall Goals: 

(i) relies on non-static assignment of plaintext ids to storage ids and accesses each storage id, and hence each encrypted object, at most once for security, This is done via timestamps $e'k = prf(k_s,ts_k')$  
  
(ii) accesses objects in batches consisting of real and fake queries on real and dummy objects, So a batch consists of:
	$f_d$: Fake queries on Fake Objects.
	$f_r$: Fake queries on Real Objects.
	$r$: Real queries on Real Objects. 


(iii) picks objects for fake queries based on the recency of access, 

(iv) caches popular objects to reduce the number of fake queries necessary to ensure obliviousness. This implies that an object either only resides in the cache or the server. 




#### Questions:


1. The proxy first generates D random dummy keys and values of the same length as of the real object. I wonder how this maps out to relational. 

2. Why are we adding a write req to dedupReqs? Is it to increment timestamp? Or to read before writing? 
3. Theorem 7.2: Overlap b/w R from reads and B-$f_d$ from writes. 
4. Whats happening with multi-threaded performance? 
5. 