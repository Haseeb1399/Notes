

January 9th 2024 (In-Person Class - 1)
<hr>

January 16th 2024 (In-Person Class - 3)


Valgrind (High Overhead in Concurrent setting)

use argument: --fair-sched=yes 


January 18th 2024 (In-Person Class - 3)

How can you have 2 LP's for a single operation? One for CAS and one for POP? Is it because it returns null and never reaches the second LP?  How do you decide which one to use?


--- 
1) What is exactly a write instead of fetch and add? is it val=val+1/val+=1 --> val.store(1)
2) Lock her subcounter is the same as a lock and then going over loop?
3) When your looping over an array of atomics, arent stale reads possible?  when x was the write thing to return, the sum was x, at that point in time. 
4) if i declare array of [64* MAX_ARRAY] i get better performance, is this cache coherence? --> Padding 
5)Difference b/w disjoint faa and waitfree is that waitfree is padded to different cache lines?
6) How do you assign a local variable in this class based implementation?
7) Does returning the leftover stuff in approx counter make the test approach to 0% off of the actual value?



_____ 

1.1.2: 
A write is a simple .store(1), so it will be a .store(v.load()+1). 
Run an experiment for this, 10 iterations each for different threads. 
Run same experiment for simple fetch and add on single address
Run same experiment for a disjoint fetch and add. 
	1) V[MAX_ARRAY]
	2) V[MAX_ARRAY] with padding. 
1) Compare and contrast which is better. 

1.1.3: 
	Run Experiment for simple fetch and add on a single address.
	Run experiment for fetch and add counter. Vary c and n and see error rates. Compare how they scale over time.

1.1.4 
1) Read operation linearizable, is it easy to do the linearizable proof using a single lock? rather than multiple different locks?



<hr>


