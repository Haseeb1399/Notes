### Section 3: Explanation:

1. **EQ**: This is a scheme that allows for equality checks using a type of searchable encryption. The goal is to allow someone to check if a word matches an encrypted value without revealing the actual word or the encrypted value.
    
2. **EQEnck(v)**: This is the encryption function. Given a value �v, it produces an encrypted output consisting of two parts:
    
    - ��IV: A random value (Initialization Vector).
    - �������(�)(��)AESKDFk​(v)​(IV): An AES encryption of the IV using a key derived from �v using the KDF (Key Derivation Function).
3. **EQTokenk(w)**: This function generates a token for a word �w. The token is generated using the KDF with the word �w.
    
4. **EQMatch((IV, x), tok)**: This function checks if a given token matches an encrypted value. It does this by decrypting the encrypted value �x using the token and checking if the result matches the IV.
    
5. The encryption is randomized (because of the random IV), so you can't build a direct index on it. If you need to search for a value, you'd have to scan through all the encrypted values (linear scan). If indexing is desired, a different scheme (ArxEq index) is used.

1. **Encryption**:
    
    - Choose a random ��IV, let's say "abcd".
    - Encrypt the code "12345" using EQEnck. This would involve deriving a key from "12345" using KDF and then encrypting "abcd" with this key. Let's say the encrypted value is "xyz".
    
    So, the encrypted entry in the database for "12345" is ("abcd", "xyz").
    
2. **Generating a Token**:
    
    - Someone wants to check if "12345" exists in the database. They use EQTokenk to generate a token for "12345". This token is the key derived from "12345" using KDF.
3. **Matching**:
    
    - The server proxy uses EQMatch to check if the token matches the encrypted entry ("abcd", "xyz"). It decrypts "xyz" using the token and checks if the result is "abcd". If it is, then it means "12345" exists in the database.

<hr>

### Notes

ArxRange: Range + Order by limit queries 
ArxEq: Equality Queries 

Range Queries: Build a tree over relevant keywords, stores at each node in the tree a garbled circuit. Tree is history-independent to reduce structural leakage. 

ArxEq builds a regular database index over encrypted values by embedding a counter into repeated values

EQ:
EQEnc(V): 
	* Take an IV (random value), generate a key from the value V using a KDF-Key derivation algorithm and compute the AES of the IV using this key. 
	* When you want to search for a word w, make a token using KDE(w), send it to server
	* Server combines calculates the AES(IV) using this token as key and see if it makes the encrypted value. 
	* EQUnique is just normal deterministic enrcyption. 


ArxRange:
Layman approach: 
* Index is a binary tree, encrypt and send to server. To locate a value a, server provides client a node (starting from the root) and client tells the server to fetch r/l node. 
* ArxRange: 
	* Create Tree on Attribute 
	* Algo: Garble, Encode, Eval
	* Can garble a boolean circuit f to F and encoding e
	* Given input a, client runs Encode(e,a)
		* Produce Ea
	* Server runs Eval (F,Ea) to obtain f(a)
	* You learn nothing about a or f. Can only use this once. If client provides two encodings Ea, Eb using same encoding information e to the server, security guarantee no longer holds. 
	* Each node must be re-encoded. 
	* Hidden function returns encoding compatible with the relevant child node. 
	* Note that since each node encodes the < operation, in order to perform ≤ operations the client needs to first transform the query into an equivalent query with the < operation; e.g., age ≤ 5 is transformed to age < 6 instead.
	* To repair garble circuit, client needs to supply garbled circuits. Only log number of circuits get consumed. For each node, client needs value encoded in N and the ID of the unused child circuit.

* Index:
	* Store at each node, encrypted primary key of the documents containing the value. Leaves are ordered, but doesnt leak order. We get encrypted IDs and send back to client, which decrypts and shuffles, then gets the ids. 
	* `Select * from patients where 1<age≤5` we find nodes 1 and 5, then traverse nodes in between to find ids. 
	* for limit, just keep record of # of ids from left to right and send those when limit exceeded. 
	* Inserts and deletes, the server traverses the index to the position, performs operation and rebalances. For updates first delete then insert. As a result of rebalancing, it is possible we need to repair the tree. (Upper bounded by height of tree)
	* Some updates and deletes may first perform filter using a different index. Backward pointers from the document to the corresponding nodes. No need to traverse the tree then. 

* ArxEq (Equality)
	* Encrypt value and tell DB o build regular index. 
	* For Unique attributes, use simle EQunique deterministic encryption. 
	* Client stores a map, called `ctr` mapping each distinct value v of attribute that exists in db to a counter. 
	* Encrypt and insert: 
		* Inserting age v. 
		* Client proxy first increments ctr[v]. 
		* $$Enc(v) = H(EQunique(v),ctr[v])$$
		* Here H is a cryptographic hash. Arx encrypts v with Base as well. Since EQ isn't decryptable.
		* Search token: 
			* Search token for a value v is the list of encryptions for every counter from 1 to ctr[v]. 
		* Search is simply `age = token1 or token2 ...`
		* Optimization: 
			* Client has to work (encrypt) ctr[v] times, we can bring this down to log ctr[v].
			* Make a stree:
				* Root is $$EQunique_k(v)$$
				* Left child contains hash of parent concatenated with 0 and right contains hash of parent concatenated with 1. 
				* Leaves correspond to counter 0,1,2,3...ctr[v]
				* When you want to search, traverse tree to find internal nodes whose subtree covers the leaf nodes 0... ctr[v]-1 exactly. Send to server, server will expand and get values. 
	* Updates:
		  * For delete: Simply delete document.
		  * For update: Delete and Insert
		  * Thus encrypted values for some counter values will not return anything. Effects throughput, also causes leakage because future search leaks how many items were deleted. 
		  * Run cleanup process after each delete. 
			  * Server tells client proxy how many matches were found for a search, say ct.
			  * Proxy updates ctr[v] to ct. Chosses a new key `K'` for v and generates new tokens using new K. Sends to server to replace the fields found matching with these. 
* ArxAgg:
	* Avg: Do SUM and Count on server, Client does division.
	* For the ArxRange index tree. At every node N in the tree, we add partial aggregate corresponding to the SubTree N. N contains partial sum of attribute corresponding to the leaves under N. Root contains the sum of all values. Stored in encrypted form. 
	* Over range [70,80], locate the edges, find the perfect covering set. Return encrypted aggregates. 
	* In case of inserts/deletes or modifying you have to do an update in the path from N to Root. 
* ArxJoin: 
	* Can do Foreign key joins. 
	* `select * from C1 Join C2 on C1.fkey = C2.ID`
	* Say you have collection C2 with primary key ID (EQunique). Collection C1 has field age with ArxEQ index and diagonsis column which is foreign key. 
	* Inserting doc into 1 (10,'Flu'):
		* C1.diagnosis is encrypted with BASE - Hide the value
		* To perform join, generate encrypted pointer for C1.diagnosis. When decrypted, this pointer will point to appropriate C2. ID. 
		* Instead of using one key for ArxEq, client now uses 2 keys. It generates token for each key and includes t1 in the document as before. It uses t2 to encrypt the diagnosis 'flu' as: $$J = Base_t (EQunique('flu'))$$
		* J helps with the join. (10,'flu') becomes: (BASE(10),t1,BASE('flu'),J). Token2 is not added to the db. 
		* To Join, server will compute t1 and t2 for the age 10, as usual with ArxEQ. It locates document of interest using t1 and then uses t2 to decrypt J and obtain EQunique('flu'). This value is primary key in C2. 
* Arx Planner:
	* ArxRange and ArxEq support compound indices. 
	* Compound ArxEQ index on (A,B) cannot be used to find equality on A alone. 
	* A regular index serves for both range and equality operations. This is not true in Arx, where we have two different indices for each operation. We choose not to use an ArxRange index for equality operations because of its higher cost and different security.
* Security: See figure
	* SP is which equality queries relate to the same keyword
	* History: Leaks the list of all write operations on keyword w (delete, insert)
* Functionality:
	* Can't support timestamps, text searches and regex. 

https://www.usenix.org/system/files/sec20-demertzis.pdf