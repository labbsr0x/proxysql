# Simple example with 3 MySQL Backends
Example adapted from (https://github.com/sysown/proxysql/wiki/ProxySQL-Configuration)

## Composing up!

First things first. Bring up all the 3 MySQL Instances and the ProxySQL Instance:
`docker-compose up -d && docker-compose logs -f`

## Sysbench preparing

As we will run a a sysbench test, we need to prepare the databases for it in each mysql instance that are part of your backend. Here are the steps:
* For each of the MySQL Instance, create a database named **dbtest**. `CREATE DATABASE dbtest`
* For each of the MySQL Instance, run the "prepare" command to create and set the **sysbench table**: 
- `docker-compose exec mysql-1 sysbench --test=oltp --oltp-table-size=10000 --mysql-db=dbtest --mysql-user=root --mysql-password=root prepare`
- `docker-compose exec mysql-2 sysbench --test=oltp --oltp-table-size=10000 --mysql-db=dbtest --mysql-user=root --mysql-password=root prepare`
- `docker-compose exec mysql-3 sysbench --test=oltp --oltp-table-size=10000 --mysql-db=dbtest --mysql-user=root --mysql-password=root prepare`

## Configuring ProxySQL

Now we will configure the backend servers, the users, read-only groups, query rules... let's do it.!
### Inserting the Backends

```
Admin> INSERT INTO mysql_servers(hostgroup_id,hostname,port) VALUES (1,'mysql-1',3306);
Query OK, 1 row affected (0.00 sec)

Admin> INSERT INTO mysql_servers(hostgroup_id,hostname,port) VALUES (1,'mysql-2',3306);
Query OK, 1 row affected (0.00 sec)

Admin> INSERT INTO mysql_servers(hostgroup_id,hostname,port) VALUES (1,'mysql-3',3306);
Query OK, 1 row affected (0.00 sec)

Admin> SELECT * FROM mysql_servers;
+--------------+----------+------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| hostgroup_id | hostname | port | status | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment |
+--------------+----------+------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| 1            | mysql-1  | 3306 | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 1            | mysql-2  | 3306 | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 1            | mysql-3  | 3306 | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
+--------------+----------+------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
3 rows in set (0.00 sec)
```


### Monitoring config

```
UPDATE global_variables SET variable_value='root' WHERE variable_name='mysql-monitor_username';
UPDATE global_variables SET variable_value='root' WHERE variable_name='mysql-monitor_password';
LOAD MYSQL VARIABLES TO RUNTIME;
SAVE MYSQL VARIABLES TO DISK;
```


### Configuring read only instance

In the MySQL instance you want to be read only, run the following query: `set global read_only = 1`
And then, in the ProxySQL admin console, run: `INSERT INTO mysql_replication_hostgroups VALUES (1,2,'cluster1'); LOAD MYSQL SERVERS TO RUNTIME;`
Wait a few seconds, and then: `SELECT * FROM monitor.mysql_server_read_only_log ORDER BY time_start_us DESC LIMIT 10;`


### Users config

```
INSERT INTO mysql_users(username,password,default_hostgroup) VALUES ('root','root',1);
LOAD MYSQL USERS TO RUNTIME;
SAVE MYSQL USERS TO DISK;
```


## Running a `sysbench` test 
Now we will run a **sysbench`** test connecting to our ProxySQL
For that, we will bring up a container that runs the test and them dies.

```
docker run --rm --link proxysql_simple_test:proxysql --net=bridge labbsr0x/mysql:5.7-sysbench sysbench --num-threads=4 --max-time=20 --debug=off --test=oltp --mysql-user='root' --mysql-password='root' --oltp-table-size=100 --mysql-host=proxysql --mysql-port=6033 --mysql-db=dbtest run
```

To check whether the statements were effective and to **which hostgroup the queries were proxied**, run in the ProxySQL Admin console: `SELECT hostgroup hg, sum_time, count_star, digest_text FROM stats_mysql_query_digest ORDER BY sum_time DESC;`

To clear these statistics, just run `SELECT * FROM stats_mysql_query_digest_reset LIMIT 1;`


### Configuring Query Rules

```
INSERT INTO mysql_query_rules (rule_id,active,username,match_digest,destination_hostgroup,apply) VALUES (10,1,'root','^SELECT c FROM sbtest WHERE id=\?$',2,1);

INSERT INTO mysql_query_rules (rule_id,active,username,match_digest,destination_hostgroup,apply) VALUES (20,1,'root','DISTINCT c FROM sbtest',2,1);

LOAD MYSQL QUERY RULES TO RUNTIME;

SELECT match_digest,destination_hostgroup FROM mysql_query_rules WHERE active=1 AND username='root' ORDER BY rule_id;
```

## Mixing all together

To see it all in action, 
1. In the ProxySQL admin console, reset the **query stats** by executing: `SELECT * FROM stats_mysql_query_digest_reset LIMIT 1;`
2. In the ProxySQL admin console, **disable** the query rules by executing: `UPDATE mysql_query_rules SET active=0;LOAD MYSQL QUERY RULES TO RUNTIME;`
3. run the *sysbench* command:
```
docker run --rm --link proxysql_simple_test:proxysql --net=bridge labbsr0x/mysql:5.7-sysbench sysbench --num-threads=4 --max-time=20 --debug=off --test=oltp --mysql-user='root' --mysql-password='root' --oltp-table-size=100 --mysql-host=proxysql --mysql-port=6033 --mysql-db=dbtest run
```
4. In the ProxySQL admin console, check the query stats by executing: `SELECT hostgroup hg, sum_time, count_star, digest_text FROM stats_mysql_query_digest ORDER BY sum_time DESC;`. You may see something like this (with all the queries proxied to the hostgroup 1):
+----+----------+------------+-------------------------------------------------------------------+
| hg | sum_time | count_star | digest_text                                                       |
+----+----------+------------+-------------------------------------------------------------------+
| 1  | 18530530 | 54414      | SELECT c from sbtest where id=?                                   |
| 1  | 11428844 | 4638       | COMMIT                                                            |
| 1  | 5478108  | 10125      | UPDATE sbtest set k=k+? where id=?                                |
| 1  | 2949915  | 5445       | SELECT DISTINCT c from sbtest where id between ? and ? order by c |
| 1  | 2747293  | 5445       | SELECT c from sbtest where id between ? and ? order by c          |
| 1  | 2544686  | 5445       | BEGIN                                                             |
| 1  | 2464485  | 4850       | UPDATE sbtest set c=? where id=?                                  |
| 1  | 2442737  | 5445       | SELECT c from sbtest where id between ? and ?                     |
| 1  | 2330577  | 5445       | SELECT SUM(K) from sbtest where id between ? and ?                |
| 1  | 1641472  | 4638       | INSERT INTO sbtest values(?,?,?,?)                                |
| 1  | 1598248  | 4638       | DELETE from sbtest where id=?                                     |
| 1  | 500      | 1          | SHOW TABLE STATUS LIKE ?                                          |
+----+----------+------------+-------------------------------------------------------------------+

5. In the ProxySQL admin console, reset again the **query stats** by executing: `SELECT * FROM stats_mysql_query_digest_reset LIMIT 1;`
6. In the ProxySQL admin console, **enable** the query rules by executing: `UPDATE mysql_query_rules SET active=1;LOAD MYSQL QUERY RULES TO RUNTIME;`
7. run **again** the *sysbench* command:
```
docker run --rm --link proxysql_simple_test:proxysql --net=bridge labbsr0x/mysql:5.7-sysbench sysbench --num-threads=4 --max-time=20 --debug=off --test=oltp --mysql-user='root' --mysql-password='root' --oltp-table-size=100 --mysql-host=proxysql --mysql-port=6033 --mysql-db=dbtest run
```
8. In the ProxySQL admin console, check the query stats by executing: `SELECT hostgroup hg, sum_time, count_star, digest_text FROM stats_mysql_query_digest ORDER BY sum_time DESC;`. You may see now something like this (with all the queries that matched the regex proxied to the hostgroup 2):
+----+----------+------------+-------------------------------------------------------------------+
| hg | sum_time | count_star | digest_text                                                       |
+----+----------+------------+-------------------------------------------------------------------+
| **2**  | 18257092 | 50314      | SELECT c from sbtest where id=?                                   |
| 1  | 10277919 | 4343       | COMMIT                                                            |
| 1  | 4813961  | 9402       | UPDATE sbtest set k=k+? where id=?                                |
| **2**  | 2689924  | 5035       | SELECT DISTINCT c from sbtest where id between ? and ? order by c |
| 1  | 2215977  | 5035       | SELECT c from sbtest where id between ? and ? order by c          |
| 1  | 2213640  | 5035       | SELECT c from sbtest where id between ? and ?                     |
| 1  | 2201888  | 5035       | BEGIN                                                             |
| 1  | 2076206  | 4511       | UPDATE sbtest set c=? where id=?                                  |
| 1  | 1978638  | 5035       | SELECT SUM(K) from sbtest where id between ? and ?                |
| 1  | 1449484  | 4343       | INSERT INTO sbtest values(?,?,?,?)                                |
| 1  | 1371385  | 4343       | DELETE from sbtest where id=?                                     |
| 1  | 681      | 1          | SHOW TABLE STATUS LIKE ?                                          |
+----+----------+------------+-------------------------------------------------------------------+

## Running the admin

`docker-compose exec proxysql mysql -u admin -padmin -h 127.0.0.1 -P6032 --prompt='Admin> '`