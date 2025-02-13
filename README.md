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


