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

if you want to log as mysql user
$ sudo usermod -s /bin/bash mysql
$ sudo su - mysql  
$ exit # for exit from mysql user to previous user   
```



