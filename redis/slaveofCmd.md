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
#define REPL_STATE_CONNECT 1 /* Must connect to master */   // start
#define REPL_STATE_CONNECTED 15 /* Connected to master */   // end

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

### set cmd backtrace
```
(gdb) bt                                                              
#0  propagate (cmd=0x748770 <redisCommandTable+80>, dbid=0, argv=0x7ffff6821280, argc=3, flags=3) at server.c:2136    
#1  0x000000000042a767 in call (c=0x7ffff6914b00, flags=15) at server.c:2315                                          
#2  0x000000000042b008 in processCommand (c=0x7ffff6914b00) at server.c:2533                                          
#3  0x000000000043ba33 in processInputBuffer (c=0x7ffff6914b00) at networking.c:1299                                  
#4  0x000000000043bd24 in readQueryFromClient (el=0x7ffff68340a0, fd=7, privdata=0x7ffff6914b00, mask=1)              
    at networking.c:1363                                                                                              
#5  0x0000000000421f41 in aeProcessEvents (eventLoop=0x7ffff68340a0, flags=3) at ae.c:412                            
#6  0x0000000000422102 in aeMain (eventLoop=0x7ffff68340a0) at ae.c:455                                                
#7  0x000000000042f27b in main (argc=1, argv=0x7fffffffe2b8) at server.c:4116     
```

in `call()`:
``` c
/* Propagate the command into the AOF and replication link */
    if (flags & CMD_CALL_PROPAGATE &&
        (c->flags & CLIENT_PREVENT_PROP) != CLIENT_PREVENT_PROP) {
            ...
            /* Call propagate() only if at least one of AOF / replication
            * propagation is needed. */
            if (propagate_flags != PROPAGATE_NONE)
                propagate(c->cmd,c->db->id,c->argv,c->argc,propagate_flags);
            ...
        }
```

