
s="bananas"
l=6

ban 
ana
nan
ana
nas
as_ 
s__

ban = [1]
ana = [2,4]
nan = [3]
nas = [5]
as_ = [6]
s__ = [7]


sorted and permuted 

ana as_  ban  nan  nas  s__  


Objective: 

Outsource sensitive strings while providing the functionality of substring search 
How can you do filtering securely? (Without the server knowing what your looking for). 

Input a query string q 
Output: The set of matching positions for q in a string 

-----

Pre-liminary: They use Frequency Hiding, Order Preserving Encryption (FHOPE)
High-level overview 

Say you want to do a range query using OPE: 
Domain is 10-20 

Enc(10) --> 8
Enc(11) --> 10
.
. 
Enc(20) --> 22

Since order is preserved we can easily generate Upper and Lower bound search tokens and get data. But we are still prone to frequency attacks.

FHOPE: 

lets say Enc(10) --> 7,8,9 (Mapped to these ciphertexts)

this way the frequency of 10 is spread across different cipher texts but maintains order, 

--------


Focus only on How it is outsourced. 

You have string "Bananas", you want to be able to do substring search over it and get the positions of where the substring starts

1. Make |s| k-grams (Specify k=3). So we will make 6 3-grams which are: 
	1. ban, ana, nan, ana, nas, as_, s__ 
	2. Make a Map that saves position against each K-gram 
		1. ban --> [1]
		2. ana --> [2,4]
	Sort the Map lexographically 
	3. For each list, permute the ordering of the positions
	4. Assign each position a FHOPE and encrypt position 
	5. Add to index FHOP - Enc(Pos)
	6. To client State, add K-gram --Start FHOP -- End FHOP 
	7. Do for all

Now you have your index and state. 

---- 
Token generation: 

when length of q < k  (For example q = 'an')
Find the last K-grams in user state that is smaller than q and first kgram that is bigger than q. (In our case its just 'ana'). Issue Range query for the Start and End FHOPE. 

When q>k, then you can break down the string into k-grams and search for them:
	1. If any kgram doesn't exist in ST, the substring doesnt occur 
	2. It is still possible the substring doesnt occur, we need to verify the ordering of positions after processing. For example Hello --> "he","el","ll","lo". If q = 'hell', we will get positions for "he" and "ll" but the positions are 1 and 3, so they arent consecutive. 


Goes over other filtering strategies but I won't go over them. 

1) Expensive
2) Complicated Updates in our case. 
3) Not entirely sure about the use case. We get positions, so we can verify the substring exists but how do we map to string retrieval. 
4) What do we set for k? 



__ _ 
Student Table: 

| StudentID | FirstName | LastName | Email                     | DateOfBirth |
|-----------|-----------|----------|---------------------------|-------------|
| 1         | Emma      | Johnson  | emma.johnson@email.com    | 2001-04-12  |
| 2         | Liam      | Smith    | liam.smith@email.com      | 2002-05-19  |
| 3         | Olivia    | Lee      | olivia.lee@email.com      | 2003-07-22  |
| 4         | Noah      | Brown    | noah.brown@email.com      | 2001-08-30  |
| 5         | Ava       | Davis    | ava.davis@email.com       | 2002-01-15  |
| 6         | Ethan     | Miller   | ethan.miller@email.com    | 2003-03-03  |
| 7         | Sophia    | Wilson   | sophia.wilson@email.com   | 2004-09-17  |
| 8         | Jacob     | Moore    | jacob.moore@email.com     | 2000-11-08  |
| 9         | Mia       | Taylor   | mia.taylor@email.com      | 2002-12-25  |
| 10        | William   | Anderson | william.anderson@email.com| 2001-10-05  |

Courses Table: 

| CourseID | CourseName         | Instructor          |
|----------|--------------------|---------------------|
| 101      | Mathematics        | Dr. Alan Turing     |
| 102      | English Literature | Prof. Emily Bronte  |
| 103      | Physics            | Dr. Marie Curie     |
| 104      | Chemistry          | Prof. Dmitri Mendeleev |
| 105      | History            | Dr. Howard Zinn     |
| 106      | Computer Science   | Prof. Ada Lovelace  |
| 107      | Biology            | Dr. Charles Darwin  |
| 108      | Economics          | Prof. Adam Smith    |
| 109      | Philosophy         | Dr. Simone de Beauvoir |
| 110      | Art History        | Prof. Vincent Van Gogh |



Consider : All tables are combined in Waffle B (We don't distinguish b/w them)

Pros: 
1) Server doesn't know which which table is more popular/accessed more often. 
2) BST for dummy object doesn't need to be maintained for each table (Less # of trees overall)
3) Hides the table sizes (Roughly), 
4) If two tables have frequent join queries maybe we can leverage some query optimizations if its unified into one big table. 


Consider : All independent table (We keep 1 waffle for 1 table)

1) Complements our indexing approach, we are already keeping per table BSTs for indexes (for point and range queries)
2) Smaller Tree sizes/heights compared to combined approach. Can result in better efficiency (Like updating all dummy object timestamps in Table 1)
3) Maybe we can allow more granular control (Per table Alpha-Beta parameters?)
4) If we keep per table caches, depending on table popularity, we can return popular values directly from cache (High Read workload, low write workload)
5) Also give user ability to store some data as encrypted and some as plaintext? (Don't encrypt everything, just part of the database) --> Not feasible if we have joins to this table coming from encrypted tables. 

---- 
Delete: 

Client: 
if op == delete : 
	ReadBatch.append (getIndex(k)) 
	Bst.Delete(k)


On getting response from Server: 

if key in delete_list: 
	writebatch <-- Dummy Object
	delete_list.remove(key)

If we have a Batch size of 100 that we are sending to server:
	1. 80 Real object requests (R + f_r + del_r)
	2. 20 Dummy Object Requests

When we get resp from server, it will have 100 keys: 
	1. If the key was in delete_list, don't evict anything from cache, 
	2. Add dummy object instead 
	3. Delete key from list. 

The write batch will now have 70 Read Objects and 30 Dummy Objects

Doesn't effect Scheme since B refers to lowerbound (we can delay writing back to the server)





