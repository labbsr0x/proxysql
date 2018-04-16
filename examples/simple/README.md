# Simple example with 2 MySQL Backends
Example adapted from (https://github.com/sysown/proxysql/wiki/ProxySQL-Configuration)

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

## Sysbench testing

To run a sysbench test, first you need to prepare the database and table for it in each mysql instance that are part of your backend. Here are the steps:
* For each of the MySQL Instance, create a database named `dbtest``
* For each of the MySQL Instance, run the "prepare" command to create and set the **sysbench table**: 
- `docker-compose exec mysql-1 sysbench --test=oltp --oltp-table-size=10000 --mysql-db=dbtest --mysql-user=root --mysql-password=root prepare`
- `docker-compose exec mysql-2 sysbench --test=oltp --oltp-table-size=10000 --mysql-db=dbtest --mysql-user=root --mysql-password=root prepare`
- `docker-compose exec mysql-3 sysbench --test=oltp --oltp-table-size=10000 --mysql-db=dbtest --mysql-user=root --mysql-password=root prepare`

### Configuring Rules

```
INSERT INTO mysql_query_rules (rule_id,active,username,match_digest,destination_hostgroup,apply) VALUES (10,1,'root','^SELECT * FROM teste',1,1);
```

## Running the admin

`docker-compose exec proxysql mysql -u admin -padmin -h 127.0.0.1 -P6032 --prompt='Admin> '`