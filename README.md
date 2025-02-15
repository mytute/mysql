# MySql for production server   

## Step 1 — Installing MySQL  

```bash
$ sudo apt update
$ sudo apt install mysql-server # install mysql server package 

# start mysql server
$ sudo systemctl start mysql # ubuntu, "mysql.service" also works 
$ sudo systemctl start mysqld # fedora
$ sudo systemctl status mysql # ubuntu
```

## Step 2 — Configuring MySQL    
by default mysql-server no root password set.   
login mysql as root with sudo privilege.    
```bash
$ sudo mysql   
```
set root password   
```bash
mysql> ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'put_your_password_here';
```
start make secure mysql   
```bash
$ mysql_secure_installation
# Change the password for root ? y
# Remove anonymous users ? y
# Disallow root login remotely ? y
# Remove test database and access to it ? y
# Reload privilege tables now ? y  
```
now you can login using following command as root or any user.(sudo mysql not works now)  
```bash
$ mysql -u root -p
```

## Step 3 — Creating a Dedicated MySQL User and Granting Privileges   

```bash
# login to mysql as root
$ mysql -u root -p

# create user with password using 'caching_sha2_password' (MySQL’s default plugin for authenticate)   
# mysql> CREATE USER 'username'@'host' IDENTIFIED WITH authentication_plugin BY 'password';
mysql> CREATE USER 'sammy'@'localhost' IDENTIFIED BY 'password';

# grant appropriate privileges for created user for a database table.  
mysql> GRANT SELECT, INSERT ON database.table TO 'username'@'host';

# grant SELECT, INSERT, and UPDATE privileges to a user for all tables in a specific database
mysql>  GRANT SELECT, INSERT, UPDATE ON mydb.* TO 'username'@'host';

# grant appropriate privileges for created user for all (database or table).   
mysql> GRANT CREATE, ALTER, DROP, INSERT, UPDATE, INDEX, DELETE, SELECT, REFERENCES, RELOAD on *.* TO 'sammy'@'localhost' WITH GRANT OPTION;

# grant priviledes for complete control over every database on the server.
mysql> GRANT ALL PRIVILEGES ON *.* TO 'sammy'@'localhost' WITH GRANT OPTION;

# see  privileges assigned to a specific user
mysql> SHOW GRANTS FOR 'username'@'host';
mysql> SHOW GRANTS; # you own privileges

# remove privileges
mysql> REVOKE INSERT, UPDATE ON school.students FROM 'samadhi'@'localhost';

# After revoking privileges, you may need to run FLUSH PRIVILEGES to ensure the changes take effect immediately
mysql> FLUSH PRIVILEGES;
```

allow remote access on VPS database   
```bash

mysql> UPDATE mysql.user SET host = '%' WHERE user = 'your_username';
mysql> FLUSH PRIVILEGES;

# allow mysql to connect from any host   
$ sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
bind-address            = 0.0.0.0 # add following line 

# connect remore mysql server from your terminal  
$ mysql -h 188.166.227.124 -u your_username -p
```

debug network connectivity  
```bash
# get program name that run on port number eg: 3306
$ sudo lsof -i :3306
#result:
#COMMAND PID  USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
#mysqld  965 mysql   25u  IPv4   8039      0t0  TCP localhost:mysql (LISTEN)

# listen port number 80 using netcat (sudo apt install netcat)
# when even port is open by ufw but not listening by anything it will give "Connection Refuse error"
$ sudo nc -l -p 80 # make listen
$ telnet 188.166.227.124 80 # check port 80

$ sudo ss -tuln | grep 3306
# result ------
# LISTEN 0      70          0.0.0.0:3306        0.0.0.0:*
# explain -----
# LISTEN: Indicates that the socket is in a listening state.
# 0.0.0.0:3306: The IP address (0.0.0.0) and port (3306) the socket is listening on. 0.0.0.0 means it is listening on all available interfaces (both IPv4 and IPv6).
# 0.0.0.0:*: The remote address and port. * means it accepts connections from any remote address.
```

##  Optimize Performance  

you can use "EXPLAIN" keyword before query for analyze queries.  
```bash
mysql> SELECT students.name, students.grade, subjects.subject_name FROM tudents JOIN subjects ON students.id = subjects.student_id;
mysql> EXPLAIN SELECT students.name, students.grade, subjects.subject_name FROM tudents JOIN subjects ON students.id = subjects.student_id;
# result
+----+-------------+----------+------------+------+---------------+------+---------+------+------+----------+--------------------------------------------+
| id | select_type | table    | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                                      |
+----+-------------+----------+------------+------+---------------+------+---------+------+------+----------+--------------------------------------------+
|  1 | SIMPLE      | students | NULL       | ALL  | PRIMARY       | NULL | NULL    | NULL |    2 |   100.00 | NULL                                       |
|  1 | SIMPLE      | subjects | NULL       | ALL  | student_id    | NULL | NULL    | NULL |    3 |    50.00 | Using where; Using join buffer (hash join) |
+----+-------------+----------+------------+------+---------------+------+---------+------+------+----------+--------------------------------------------+
```

Common Causes of Slow Queries  
1. Missing Indexes: Queries that perform full table scans.  
2. Large Data Sets: Filtering or sorting large datasets without optimizations.  
3. Complex Joins: Joins on large tables with insufficient indexing.  
4. Inefficient Queries: Using subqueries instead of joins, selecting more columns than needed, etc.  
5. Lock Contention: Concurrent updates or inserts causing row or table locks.

enable the Slow Query Log(should not use production because it shlow down mysql)   

```bash
mysql> show variables like '%slow%';

+---------------------------+-----------------------------------+
| Variable_name             | Value                             |
+---------------------------+-----------------------------------+
| log_slow_admin_statements | OFF                               |
| log_slow_slave_statements | OFF                               |
| slow_launch_time          | 2                                 |
| slow_query_log            | OFF                               |
| slow_query_log_file       | /var/lib/mysql/server-slow.log    |
+---------------------------+-----------------------------------+

mysql> show variables like '%long_query%';
+-----------------+----------+
| Variable_name   | Value    |
+-----------------+----------+
| long_query_time | 5.000000 |
+-----------------+----------+

# change the long query time to whatever you want. Queries taking more than this will be captured in the slow query log.
mysql> SET GLOBAL long_query_time = 2.00;
mysql> SET SESSION long_query_time = 1; # if above "GLOBAL" query didn't work use this cmd.   

# witch on the slow query log.
mysql> set global slow_query_log = 'ON';
mysql> flush logs;

# if you want to change 'slow_query_log_file' location you can do it inside '/var/log/mysql/filename.log'
# if you want to put it another location then need to change apparmor showing inside debug section.  
```

restart MySQL   
```bash
$ sudo systemctl restart mysql
```

you can view slow queries using   
```bash
$ tail -f server-slow.log
$ grep 'Time: 160411.*' server-slow.log | cut -c2-18 | uniq -c # find unique quires   
```
test show query log   
```bash
USE test; -- or any database

CREATE TABLE test_slow_query (
  id INT AUTO_INCREMENT PRIMARY KEY,
  data VARCHAR(255)
);

-- Insert a large number of rows to make the query slower
INSERT INTO test_slow_query (data) 
SELECT REPEAT('A', 255) FROM information_schema.tables LIMIT 100000;

-- Execute a query with inefficient operations to simulate slowness
SELECT SLEEP(2), COUNT(*) FROM test_slow_query;   
```
test on log file    
```bash
$ sudo tail -f /var/log/mysql_slow.log
```

debug    
```bash
# set log permission file permission for mysql user
$ sudo mkdir /var/log/mysql
$ sudo chown mysql:mysql /var/log/mysql

$ groups mysql # check added permission group of mysql group
$ sudo usermod -aG syslog mysql # add mysql to syslog group
$ sudo systemctl restart mysql # it required to

$ sudo systemctl status apparmor
# verify AppArmor profiles:
$ sudo aa-status
# if MySQL is restricted, update the AppArmor profile:
$ sudo nano /etc/apparmor.d/usr.sbin.mysqld
/var/log/mysql_slow.log rw, # add this line
$ sudo systemctl reload apparmor

# if you want to log as mysql user
$ sudo usermod -s /bin/bash mysql
$ sudo su - mysql  
$ exit # for exit from mysql user to previous user   
```

## Monitoring & Logging 

## High Availability & Scaling  
Replication allows a copy of data from the master database to be automatically replicated to one or more slave databases. This improves availability, disaster recovery, and read performance.

We will configure the Master VPS and Slave VPS for replication.  
* Master VPS: master_ip  
* Slave VPS: slave_ip  
* MySQL Version: Both VPS servers must have the same version installed.

### 1. Master VPS Configuration  
configuration to enable binary logging and set a unique server ID  
> /etc/mysql/mysql.conf.d/mysqld.cnf
```bash
# add/modify the following lines under [mysqld] 
log_bin = /var/log/mysql/mysql-bin.log  # Enable binary logging
server-id = 1                           # Unique server ID for the Master
```
save and restart mysql server  
```bash
$ sudo systemctl restart mysql   
```
create a replication user   
```bash
# login as root user
$ mysql -u root -p

# this user for slave to connect master. there for we need to create this user on master mysql (vps)    
mysql> CREATE USER 'replica_user'@'slave_ip' IDENTIFIED BY 'replica_password';
# add replication slave privileges to above user  
mysql> GRANT REPLICATION SLAVE ON *.* TO 'replica_user'@'slave_ip'; 
mysql> FLUSH PRIVILEGES;
```
retrieve the current binary log file and position.   
```bash
mysql> SHOW MASTER STATUS; # you need to notedown 'File' and 'Position' to enter on slave vps  
# result :
+------------------+----------+
| File             | Position |
+------------------+----------+
| mysql-bin.000001 | 154      |
+------------------+----------+
```

### 1. Slave VPS Configuration  

> sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```bash
server-id = 2                                  # Unique server ID for the Slave
relay-log = /var/log/mysql/mysql-relay-bin.log # Relay log file path
# # Relay log file path will store logs received from the Master
```
save and restart MYSQL server   
```bash
$ sudo systemctl restart mysql
```
connect from Slave to Master   
```bash
mysql> CHANGE MASTER TO MASTER_HOST='master_ip', MASTER_USER='replica_user', MASTER_PASSWORD='replica_password', MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=157;
# master_ip: IP address of the Master VPS.
# replica_user: Replication user created on the Master (created on master vps for salve loggin).
# replica_password: Password for the replication user(created on master vps for salve loggin).
# MASTER_LOG_FILE and MASTER_LOG_POS: Use the values from "mysql> SHOW MASTER STATUS" on the Master.
```

start slave and check status of connection slave-master 
```bash
# start slave   
mysql> START SLAVE;

# verify connection
mysql> SHOW SLAVE STATUS\G;
# result should be
# Slave_IO_Running: Yes
# Slave_SQL_Running: Yes
```

if you want to reset Master data in Slave VPS( "CHANGE MASTER TO ... etc" command )
```bash
mysql> STOP SLAVE; # first need to stop slave 
# reset the Slave's replication settings to clear the previously entered CHANGE MASTER TO data.
mysql> RESET SLAVE ALL;
# re-enter master-slave connection query
mysql> CHANGE MASTER TO MASTER_HOST='master_ip', MASTER_USER='replica_user', MASTER_PASSWORD='replica_password', MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=157;
mysql> START SLAVE;
```
### Verify the Replication   
create a test database and table on Master vps :   
```bash
CREATE DATABASE test_db;
USE test_db;
CREATE TABLE test_table (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(50));
INSERT INTO test_table (name) VALUES ('Replication Test');
```
check on Slave vps : 
```bash
SHOW DATABASES;
USE test_db;
SELECT * FROM test_table;
```

depend on your senerio you can add firewall configuraiton (optional)   
```bash
# On Master VPS
$ sudo ufw allow from slave_ip to any port 3306  
# On Slave VPS
$ sudo ufw allow from master_ip to any port 3306  
```
This replication works when slave gone offline and master update while slave offline will sync after slave came online.  

## Scaling and Load Balancing with ProxySQL    

Why Use ProxySQL?  
1. Load Balancing: Distributes read-heavy workloads across multiple replicas (Slaves).  
2. Write Routing: Ensures that write operations go to the Master database.  
3. Failover Handling: Redirects traffic to healthy replicas automatically in case of failure.  
4. Improved Scalability: Allows applications to scale by offloading reads from the Master to the Slaves.

install ProxySQL   
```bash
$ sudo apt update   
$ sudo apt install proxysql  # need to install on seperate vps or where you application run on.  
```




