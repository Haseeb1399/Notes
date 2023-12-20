
#### Linear Notes: 

Intro: 

SSE Trade off --> Reveal some well-defined information (leakage) about the queries and underlying data in exchange for efficiency. 

Existing schemes only support range queries on single-attribute data. 

Extend work of Demertzis and Faber to multi-attribute range queries. 

Decomposing the multi-dimensional domain _D_ into sub-ranges. Each subrange can be mapped to the corresponding set of records with values in that subrange. Use parallelization and updates are supported by batching updates. 

Opt for data-independent data structures. 

### Prior and Related Works

Demertzis and Faber: Build binary tree on domain, build an index based on this tree, then use a block-box sse scheme to encrypt index. 

### Contribution 

Extend standard quadtree to noval tree data structure. Prove reduction in # of false positives. 

General frameowkr for building private range searc schemes. 
Reduce the range query to a set of queries on on underlying encrypted multimap. 6 range schemes. You can choose between efficiency and security tradeoffs. 

These range-search data structures are best used for a small # of attributes and even in the plaintext setting, alternative approaches (project to one-dimension and filter the results)
should be used to handle higher dimensional data. 


Range covering methods:
Reduce range query --> to a set of precomputed queries --> answers stored in multimaps. 

Identify form of leakage called structure pattern: Pattern of co-occurance of subqueries. For example, Queries from patient records and medicine records being done together. 

### Preliminaries 

**𝑑-attribute database or 𝑑-dimensional database, 𝐷:** This is a database that can be thought of as a multi-dimensional grid or table. Each dimension ranges from 1 to some positive integer 𝑚𝑖. The total number of dimensions is 𝑑.

- The domain of this database, denoted as D, is the Cartesian product of the ranges of each dimension. So, if you have two dimensions where the first ranges from 1 to 3 and the second ranges from 1 to 2, the domain D would be {(1,1), (1,2), (2,1), (2,2), (3,1), (3,2)}.
    
- Each unique combination of values from the dimensions (each point in the grid) maps to a set of records. This mapping is injective, meaning each combination maps to a unique set of records.
    
- The size of the set of records for any domain value is of order 𝑂(1), which means it's constant and doesn't grow with the size of the domain.

###### Example: 
Imagine a school database where each student is uniquely identified by two attributes:

1. **Grade** (from 1 to 3)
2. **Class** (from A to B)

This gives us a 2-dimensional grid:

||A|B|
|---|---|---|
|**1**|||
|**2**|||
|**3**|||

Each cell in this grid represents a unique combination of Grade and Class.

Now, let's populate this grid with data:

1. Students in Grade 1, Class A: {Alice, Bob}
2. Students in Grade 1, Class B: {Charlie}
3. Students in Grade 2, Class A: {David, Eve}
4. Students in Grade 2, Class B: {Frank}
5. Students in Grade 3, Class A: {Grace}
6. Students in Grade 3, Class B: {Hannah, Ian}

Our grid now looks like this:

||A|B|
|---|---|---|
|**1**|Alice, Bob|Charlie|
|**2**|David, Eve|Frank|
|**3**|Grace|Hannah, Ian|

#### Injective Mapping:

Each unique combination (cell in the grid) maps to a unique set of students. For example, the combination (Grade 1, Class A) maps to the set {Alice, Bob}. No other combination will map to this exact set. This is the injective property.

#### Order 𝑂(1) Size:

Notice that the number of students in each cell (the set size) varies. Some cells have one student, some have two. However, the size of these sets doesn't grow with the size of the domain. In other words, even if we had 100 grades and 100 classes, the number of students in any given cell wouldn't necessarily be 10,000 or any large number. It would still be a small, constant number, like 1, 2, or 3. This is what is meant by the size being of order 𝑂(1). It's constant and doesn't grow with the size of the domain.

#### Dyadic range: 
![[Pasted image 20231022213859.png]]

