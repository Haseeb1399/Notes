Link:https://people.csail.mit.edu/stavrosp/papers/sigmod2016/SIGMOD16_RSSE.pdf

**Searchable Symmetric Encryption**: You have a range of documents in a remote server (encrypted) and you can search for a specific document within the encrypted Database without having to decrypt the entire database. Lets you find information in encrypted data while keeping data safe. 
<hr>

**Order Preserving Encryption** (OPE): Cipher-text preserves the ordering of the plain texts. Example:

[2,3,5,9,1,3] -- Encrypt --> [X,Y,Z,A,B,C]

In this scheme we still know Y>X and A>Z and A>B.
This is still prone to distribution leakage (deterministic).
<hr>

The _basic scheme_ [Fast Range Query](https://www.cse.msu.edu/~alexliu/publications/RuiRangeNonAdaptive/RuiRangeNonAdaptive_VLDB14.pdf) assumes a binary tree over the query attribute domain and for every tuple _d_ in the Dataset _D_ computes the log m dyadic ranges covering its attribute values. For example:
* Attribute is Book_Publish_Year. It will split it the total range (2000-2023) into (2000-2005) (2006-2010) and (2011-2023). 
They then make the binary tree as follows:
* Example: Let's say the root node initially contains books published from 2000 to 2023. It might randomly split this range into two child nodes: [2000-2011] and [2012-2023]. Each child node would store a Bloom filter summarizing the dyadic ranges within that node.

Query Processing: To answer a range query, the scheme splits the query range into its minimal dyadic ranges (O(logR) of them, where R is the range size). Then, it traverses the binary tree, checking whether the Bloom filters at each node "contain" any of the minimal dyadic ranges.

Example: If you want to find books published between 2015 and 2020, the scheme would split this range into minimal dyadic ranges like [2015-2016], [2017-2018], and [2019-2020]. It would then traverse the tree, checking whether the Bloom filters at each node contain any of these minimal dyadic ranges.
<hr>

Delegatable Pseudorandom Function (DPRF)

In summary, a Delegatable Pseudorandom Function (DPRF) enhances the capabilities of a PRF by allowing the owner of the secret key to delegate the computation (without knowing the secret key _k_) of DPRF values for a subset of the domain to another party. This delegation improves performance by distributing the computational load, while still maintaining the security properties expected of a PRF. 

<hr>

### Quadratic Scheme : It's a naive baseline.

A --> Query Domain (All possible values of an attribute. Say Age)
m --> Size of Domain (A) 

This scheme will enumerate all possible subranges of A $O(m^2)$ and assign a unique keyword to each range. Then associate each data point with the keywords corresponding to the ranges. (A single data point can have multiple keywords since ranges can be overlapping).

Cons: 
Very slow and very large index size.
Replication increases database size. 

<hr> 

### Constant Scheme: 
From what I understand, they build index using DPRF instead of simple PRF. This allows them to build a GGM Tree where they can use range covering algorithms to reduce the query size to $O(log R)$. But this does have better storage $O(n)$ compared to $O(m^2)$ in the quadratic scheme. 

GGM Trees are used for efficient range queries. 

Note that is isn't safe to adversaries that allow issuing intersecting range queries. A lot of structural leakage in this scheme. 

1. - If the first query is **w1: [0, 3]** and it returns documents **d1** and **d2** with **d1.a=0** and **d2.a=3**:
        - The leakage **L2** for this query would include an alias for a node covering the range [0, 3], its level in the tree, and mapping info that shows **d1** corresponds to the left-most leaf and **d2** to the right-most leaf.
    - If there's a second query **w2 = [5,7]**:
        - The leakage would additionally include aliases for nodes covering this range and the mapping of qualifying tuples in these subtrees.
        - Notably, the order of **w1** and **w2** in **A** or the relative order of the subtrees for each query isn't disclosed, ensuring some level of privacy.


<hr>

### Logarithmic Scheme 

1. Doesn't use DPRF 
2. Associating each data tuple with a logarithmic number of keywords rather than a constant number. 

BuildIndex: 
* Builds a binary tree over domain A. Each node is given a unique keyword
* For each tuple _d_ in Dataset **_D_**, find the nodes on the path from the root of the tree to `d.a` 
* Associate duple _d_ with the keywords of these nodes.
* Replicate every d for each keyword it is associated with. 
* Treat each copy of D as a separate tuple. 
* Apply standard BuildIndex algorithm of SSE to _**D'**_ 

Example: 
          N0,7
         /     \
      N0,3     N4,7
     /   \    /     \
  N0,1  N2,3  N4,5   N6,7
  / \  /   \ /   \   /   \
N0 N1 N2 N3 N4 N5 N6 N7

For simplicity, let's use the node labels as keywords. For instance, the keyword for N0,3 will be "N0,3".

$d_1.a = 3$ lies in the path N0,7 --> N0,3 --> N2,3 --> N3
keywords: (N0,7),(N0,3),(N2,3),(N3)

d1 will be replicated 4 times. New dataset will be created with 
- d1​ with keyword N0,7
- d1​ with keyword N0,3
- d1​ with keyword N2,3
- d1​ with keyword N3

#### Token Generation: 
- **Operation**: t←Trpdr(k,w)
    
- The range of interest in the query is represented by w.
    
- Identify the nodes that cover this range using either BRC or URC.
    
- For each of these nodes, generate a token using SSE's Trpdr algorithm.
    
- These tokens are collectively represented by t.
    
- Before returning t, the tokens are randomly permuted.
    
- **Example**: For a query range [2,7][2,7] using BRC, this step would produce SSE tokens for nodes N2,3 andN4,7.

##### Cost: 

1. Index Size: O(n log m) since n is the number of tuples in the dataset and m is the domain size 
2. Query Size: O(log R) since R is the size of the query. 
3. Search Time O(Log R + r) where r is the result set. Log R factor comes from the number of tokens, while r represents the time for actual data retrieval. 

<hr>

### Logarithmic - SRC (Single Range Cover)

Leaking in Logarithmic scheme was because of Trpdr producing multiple tokens, one for each subtree with which BRC or URC cover the query range. 

Logarithmic-SRC, prevents this leakage by always covering the query range with a sin- gle range that is potentially a superset of the query range. In addition to enhanced privacy, this also leads to constant query size, but may in- troduce false positives.

The RSSE algorithms for Logarithmic-SRC are the same as in Logarithmic-BRC/URC with the following differences: (i) in BuildIndex, each d ∈ D is associated with the key- words/labels of the nodes of TDAG that cover d.a (instead of the nodes of the binary tree), and (ii) Trpdr generates a single token for the node label output by the SRC covering technique (instead of BRC/URC).

### Logarithmic - SRC-i 

Double indexing to reduce number of false positives, but requires more communication. 


<hr> 

<hr> 

### Misc:

What is R? (Size of query range)

For instance, if you want to know about all tuples whose associated value (attribute) lies between 5 and 10, your range size �R is 10−5+1=610−5+1=6.

<hr> 
### **Questions**: 

1. In BRC, for a range R, there are O(log R) nodes. For the example [1,6] we can see they use 4 nodes which is more than O(log(6)). Why?
2. What auxiliary information allows for server to partially decrypt? What's partial decryption? 
3. Still don't completely understand the privacy benefits of logarithm brc/urc in comparison to constant. 

<hr> 

### Comments 

1. They work on a single attribute, we can have range queries over multiple attributes (AND relations), will be hard to maintain an index for all columns
2. It would be nice to see what part of obliviousness we can delegate to Waffle and what part can be delegated to indexing. Don't have to make indexing responsible for everything. 
3. What about non-numeric/non-datetime attributes. 



Friday Meeting: 
Go over paper again and answer all queries made in this meeting.

1) GGM Tree --> Do we construct over entire domain or only what we have seen in the database. 
2) Do they support updates (GGM And This paper). 


__ 
https://dl.acm.org/doi/pdf/10.1145/6490.6503

https://dl.acm.org/doi/10.1145/2508859.2516668

https://dl.acm.org/doi/10.1145/3514221.3517868

https://arxiv.org/pdf/1312.4012.pdf


Find more recent private indexing papers (See ones that reference Ionize's paper). 
Present two papers that best fit with the idea. 

