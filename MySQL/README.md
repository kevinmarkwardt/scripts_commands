# MySQL Commands and Scripts

## Commands

**Skip slave event**
Make sure the event skipped isn't going to cause data loss

```sql
SET @@default_master_connection="channel_name"; # Only if you are using multi master replication.  If not skip this
STOP  SLAVE;
SET GLOBAL sql_slave_skip_counter = 1;
START SLAVE;
SHOW SLAVE STATUS\G;
```

**Speed up replication/query execution**

```sql
SET GLOBAL sync_binlog=0;
SET GLOBAL innodb_flush_log_at_trx_commit=2;
```

**Old School Replication Setup**

On Master
- Get position
- Set user account for replication

```sql
SHOW MASTER STATUS\G  # Get master position
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%' IDENTIFIED BY 'REPLICATION_PASSWORD';
FLUSH PRIVILEGES;
```

On Slave

```sql
CHANGE MASTER TO
     MASTER_HOST = '192.168.0.112',
     MASTER_USER = 'repl',
     MASTER_PASSWORD = 'REPLICATION_PASSWORD',
     MASTER_LOG_FILE = 'mysql_binlog.000001',
     MASTER_LOG_POS = 120;

START SLAVE;
SHOW SLAVE STATUS\G
```

**Process List Commands**

```sql
watch 'mysql -e "show full processlist" | egrep -v "Sleep|processlist|binlog" | sort -k6n'

SELECT * FROM INFORMATION_SCHEMA.PROCESSLIST WHERE COMMAND <> 'Sleep' ORDER BY TIME DESC LIMIT 1;

mysql -e "SHOW PROCESSLIST" | awk '{print $3}' | cut -f1 -d: | sort | uniq -c | sort -nr | awk '{print $1"\t"$2}'  # Get counts of the connections
```

**Database Sizes**

```sql
SELECT table_schema AS "Database name", SUM(data_length + index_length) / 1024 / 1024 AS "Size (MB)" FROM information_schema.TABLES GROUP BY table_schema;
```

**Top Table Sizes**

```sql
SELECT TABLE_SCHEMA, TABLE_NAME, ENGINE, DATA_LENGTH/1024/1024/1024 as DATA_LENGTH_IN_GB,
  INDEX_LENGTH/1024/1024/1024 AS INDEX_LENGTH_IN_GB,
  (DATA_LENGTH+INDEX_LENGTH)/1024/1024/1024 AS TOTAL_LENGTH_IN_GB
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('mysql', 'information_schema', 'performance_schema')
  AND TABLE_TYPE='BASE TABLE'
ORDER BY TOTAL_LENGTH_IN_GB DESC
LIMIT 10;
```

**Table Fragmentation**

```sql
SELECT concat(table_schema,'.',table_name) as Database_Tablename, concat(round(table_rows/1000000,2),'M') ROWS, concat(round(data_length/(1024*1024*1024),2),'G') DATA_SIZE, concat(round(index_length/(1024*1024*1024),2),'G') INDEX_SIZE, concat(round((data_length+index_length)/(1024*1024*1024),2),'G') TOTAL_SIZE, round(index_length/data_length,2) idxfrac  FROM information_schema.TABLES ORDER BY data_length+index_length;
```

**Config Editor**

```bash
mysql_config_editor set --host=localhost --user=kmarkwardt --socket=/mysqldata/mysql/data/mysql.sock --port=3306 --password
mysql  # Login 

mysql_config_editor set --login-path=kmarkwardt --host=localhost --user=kmarkwardt --socket=/mysqldata/mysql/data/mysql.sock --port=3306 --password
mysql --login-path=kmarkwardt  # Login using path
```

**Binary Log Cleanup Script**
Used this when disk space was filling up due to the slave being very far behind and the binary logs were filling up space

```bash
#!/bin/bash

PERC_LIMIT=80

# Will monitor the disk space and remove old binary logs to help keep the drive from filling up.
echo "Starting binary log space monitoring"
while [ 1 ]; do
        USED_PERC=`df -h | grep binlog0 | awk '{print $5}' | sed 's/%//'`
        if [ $USED_PERC -gt $PERC_LIMIT ];then
                echo "Disk space $USED_PERC has exceeded the limit of $PERC_LIMIT.  Performing Cleanup"
                BINARY_LOG=`mysql -BN -e "SHOW BINARY LOGS" | awk '{print $1}' | head | tail -n1`

                echo "Purging binary logs to $BINARY_LOG"
                mysql -e "PURGE BINARY LOGS TO '$BINARY_LOG';"

                USED_PERC=`df -h | grep binlog0 | awk '{print $5}' | sed 's/%//'`
                echo "Used disk space % is now $USED_PERC"
        fi
        sleep 5
done
```

**Get new connection per second counts and MySQL execution time**
Had a problem with a server where the connection time was being impacted and the number of connections on the box was fluctuating greatly.  Used this to track some of the details
```bash
#!/bin/bash
LAST_COUNT=0
SLEEP_TIME=$1
while [ 1 > 0 ]; do
     date
     START_TIME=`date +%s`
     mysql -e "SHOW STATUS LIKE 'connections'"
     END_TIME=`date +%s`
     RUNTIME=$((END_TIME-START_TIME))
     echo "MySQL Execution Time : $RUNTIME sec."
     CONN_COUNT=`mysql -e "SHOW STATUS LIKE 'connections'" | grep Connections | awk '{print \$2}'`
     DIFF_COUNT=$((CONN_COUNT - LAST_COUNT))
     TOTAL_EXECUTION_TIME=$((SLEEP_TIME+RUNTIME))
     echo "Total Execution Time : $TOTAL_EXECUTION_TIME sec."
     PER_SEC=$((DIFF_COUNT / TOTAL_EXECUTION_TIME))
     echo "Difference in Connections : $DIFF_COUNT ($PER_SEC / sec)"
     LAST_COUNT=$CONN_COUNT
     sleep $SLEEP_TIME
done
```