# slaveof cmd

## slave's view

### 1. client CMD handler of `slaveof`

``` c
void slaveofCommand(client *c)	// nearly do nothing about the dirty work
								// check if slaveof no one, or slaveof the current master again
								// if not, call 

void replicationSetMaster(char *ip, int port)	// set new master ip/port, set an important flag:
    server.repl_state = REPL_STATE_CONNECT;
```

### 2. timely check flag and do the dirty work 

```
(gdb) bt                                                                                                                    
#0  replicationCron () at replication.c:2239                                                                                 
#1  0x0000000000427de4 in serverCron (eventLoop=0x7ffff68340a0, id=0, clientData=0x0) at server.c:1274                       
#2  0x0000000000421c0c in processTimeEvents (eventLoop=0x7ffff68340a0) at ae.c:322                                        
#3  0x0000000000421f52 in aeProcessEvents (eventLoop=0x7ffff68340a0, flags=3) at ae.c:423                                
#4  0x00000000004220a2 in aeMain (eventLoop=0x7ffff68340a0) at ae.c:455                                             
#5  0x000000000042f226 in main (argc=1, argv=0x7fffffffe168) at server.c:4118  
```

``` c
void replicationCron(void)	// called 1 time per sec, which check the flag:
    if (server.repl_state == REPL_STATE_CONNECT)	// then call

int connectWithMaster(void)	// create a file event by calling aeCreateFileEvent, with a specific aeFileProc: syncWithMaster, and set
    server.repl_state = REPL_STATE_CONNECTING;	
	aeCreateFileEvent(server.el,fd,AE_READABLE|AE_WRITABLE,syncWithMaster,NULL)

/********** not in 1-sec-check now, fetched from aeMain **********/
void syncWithMaster(aeEventLoop *el, int fd, void *privdata, int mask)	// the really deal here, ton of work done. a new file event created, and set
    server.repl_state = REPL_STATE_TRANSFER;
    if (slaveTryPartialResynchronization(fd,0) == PSYNC_WRITE_ERROR)	// try psync, which should failed, but issue the psync-cmd still,
    																	// which will turn into a fullsync-cmd in the server side
	aeCreateFileEvent(server.el,fd, AE_READABLE,readSyncBulkPayload,NULL)	

void readSyncBulkPayload(aeEventLoop *el, int fd, void *privdata, int mask)	// do the file transfer work, and at last call

void replicationCreateMasterClient(int fd)	// set flag
    server.repl_state = REPL_STATE_CONNECTED;
```

#### how to know the sync has been finished?

``` c
#define REPL_STATE_CONNECT 1 /* Must connect to master */

struct redisServer {
	...
    int repl_state;          /* Replication status if the instance is a slave */
    off_t repl_transfer_size; /* Size of RDB to read from master during sync. */
    off_t repl_transfer_read; /* Amount of RDB read from master during sync. */
    ...
}
```

## master's view

### sync cmd
```
(gdb) bt                                                                                                                    
#0  syncCommand (c=0x7fffef0e7b40) at replication.c:570                                                                      
#1  0x000000000042a421 in call (c=0x7fffef0e7b40, flags=15) at server.c:2252                                                 
#2  0x000000000042afb3 in processCommand (c=0x7fffef0e7b40) at server.c:2535                                                 
#3  0x000000000043b9de in processInputBuffer (c=0x7fffef0e7b40) at networking.c:1299                                         
#4  0x000000000043bccf in readQueryFromClient (el=0x7ffff68340a0, fd=6, privdata=0x7fffef0e7b40, mask=1) at networking.c:1363                                                                                          
#5  0x0000000000421ee1 in aeProcessEvents (eventLoop=0x7ffff68340a0, flags=3) at ae.c:412                                  
#6  0x00000000004220a2 in aeMain (eventLoop=0x7ffff68340a0) at ae.c:455                                                      
#7  0x000000000042f226 in main (argc=1, argv=0x7fffffffe168) at server.c:4118 
```

### cmd backlog
```
(gdb) bt
#0  replicationFeedSlaves (slaves=0x7ffff681c090, dictid=-1, argv=0x7fffffffde30, argc=1) at replication.c:173              
#1  0x0000000000448b1e in replicationCron () at replication.c:2302                                                        
#2  0x0000000000427de4 in serverCron (eventLoop=0x7ffff68340a0, id=0, clientData=0x0) at server.c:1274                
#3  0x0000000000421c0c in processTimeEvents (eventLoop=0x7ffff68340a0) at ae.c:322                                         
#4  0x0000000000421f52 in aeProcessEvents (eventLoop=0x7ffff68340a0, flags=3) at ae.c:423                             
#5  0x00000000004220a2 in aeMain (eventLoop=0x7ffff68340a0) at ae.c:455                                                      
#6  0x000000000042f226 in main (argc=1, argv=0x7fffffffe168) at server.c:4118
```

``` c
void replicationFeedSlaves(list *slaves, int dictid, robj **argv, int argc) 
```






