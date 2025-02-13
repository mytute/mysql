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
mysql> CREATE USER 'username'@'host' IDENTIFIED WITH authentication_plugin BY 'password';

grant appropriate privileges for created user for a database table.  
mysql> GRANT PRIVILEGE ON database.table TO 'username'@'host';

grant appropriate privileges for created user for all (database or table).  
mysql>  GRANT PRIVILEGE ON * TO 'username'@'host';

grant appropriate privileges for created user for all (database or table).   
mysql> GRANT CREATE, ALTER, DROP, INSERT, UPDATE, INDEX, DELETE, SELECT, REFERENCES, RELOAD on *.* TO 'sammy'@'localhost' WITH GRANT OPTION;

grant priviledes for complete control over every database on the server.
mysql> GRANT ALL PRIVILEGES ON *.* TO 'sammy'@'localhost' WITH GRANT OPTION;

remove all "CREATE USER" and "GRANT" statements
mysql> FLUSH PRIVILEGES;
```




