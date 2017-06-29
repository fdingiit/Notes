# problem set

## sync problem
1. how to confirm a `sync` process has been successfully finished?
    - is the slave's data same with master's?




2. how to manully sync data when the master-slave relationship has been set up?
    - `slaveof` cmd is useless 

``` c
/**
 * a new user command, eg. RDB, to replace `slaveof` for sync, which:
 *      1. reuse server-slave tcp connection: `fd` if exists;
 *      2. create a new syncWithMaster file event.
 */
int connectWithMaster(void) {
    int fd;

    fd = anetTcpNonBlockBestEffortBindConnect(NULL,
        server.masterhost,server.masterport,NET_FIRST_BIND_ADDR);
    if (fd == -1) {
        serverLog(LL_WARNING,"Unable to connect to MASTER: %s",
            strerror(errno));
        return C_ERR;
    }

    if (aeCreateFileEvent(server.el,fd,AE_READABLE|AE_WRITABLE,syncWithMaster,NULL) ==
            AE_ERR)  // HERE!!!!
    {
        close(fd);
        serverLog(LL_WARNING,"Can't create readable event for SYNC");
        return C_ERR;
    }

    server.repl_transfer_lastio = server.unixtime;
    server.repl_transfer_s = fd;
    server.repl_state = REPL_STATE_CONNECTING;
    return C_OK;
}
```

### new thoughts:
use redis-cli linux-shell command (not in-client command):
`./redis-cli -h 172.16.82.190 -p 6379 --rdb ./123`

