

- [ ] Make waffle handle keys not present in the BST
- [ ] Waffle should do everything in a put batch and respond back with k:v pairs. 
- [ ] In plaintext KV, make sure to handle for duplicate keys. 




---

Tasks Today
- [ ] Make Executor Proxy (Sync Function, with non blocking)
- [ ] Have the executor only have an executeBatch interface
	- [ ] Get sent R requests. Read them and put them back again (all R). 
	- [ ] Send Back only Get request responses. 
- [ ] Make Load Balancer Connect to the Executor. Call the executeBatch interface (get and Put requests)
- [ ] Expose a `addKeys` interface from the load balancer. Pools requests and sends to balancer. 