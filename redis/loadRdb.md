# load RDB file

``` c
int rdbLoad(char *filename);
```

``` c
/* Function called at startup to load RDB or AOF file in memory. */
void loadDataFromDisk(void) {
    long long start = ustime();
    if (server.aof_state == AOF_ON) {
        if (loadAppendOnlyFile(server.aof_filename) == C_OK)
            serverLog(LL_NOTICE,"DB loaded from append only file: %.3f seconds",(float)(ustime()-start)/1000000);
    } else {
        if (rdbLoad(server.rdb_filename) == C_OK) {
            serverLog(LL_NOTICE,"DB loaded from disk: %.3f seconds",
                (float)(ustime()-start)/1000000);
        } else if (errno != ENOENT) {
            serverLog(LL_WARNING,"Fatal error loading the DB: %s. Exiting.",strerror(errno));
            exit(1);
        }
    }
}
```

## want a `rdb host port file` cmd




