
### Goal: 

Design a new relational database system that provides protection against access pattern and volume based attacks in Client-Server data storage model. Use Waffle as a foundation for hiding access patterns and volume when fetching or writing data to external server. System should support a variety of SQL queries such as Select, Insert, Update, Join and Where. We also need to support range queries over numeric data.

### Background: 

Waffle introduced 𝛼, 𝛽 uniformity. In summary (within the context of a key/value database), this uniformity states that:

1. 𝛼 is an upper bound that ensures any key that is written to the server, will be read after at most 𝛼 server accesses. As an example, if 𝛼 = 5, this means that every key present on the server must be accessed within 5 server accesses. 
2. 𝛽 is the lower bound that ensures any key that has been read from the server, will be written back to the server after 𝛽 accesses. As an example if 𝛽=10. A key that has been read from the server will have to wait a minimum of 10 server accesses to eligible for writing back to server. 

Using this foundation, we want to construct an database that supports SQL queries. 
### System Design 

